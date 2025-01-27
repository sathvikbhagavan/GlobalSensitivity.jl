# Using the Shapley method in case of correlated inputs

One of the primary drawbacks of typical global sensitivity analysis methods is their
inability to handle correlated inputs. The Shapley method is one of the few methods
that can handle correlated inputs. The Shapley method is a game-theoretic approach
that is based on the idea of marginal contributions of each input to the output.

It has gained extensive popularity in the field of machine learning and is used to
explain the predictions of black-box models. Here we will use the Shapley method
on a Scientific Machine Learning (SciML) model to understand the impact of each
parameter on the output.

We will use a Neural ODE trained on a simulated dataset from the Spiral ODE model.
The Neural ODE is trained to predict output at a given time. The Neural ODE is
trained using the [SciML ecosystem](https://sciml.ai/).

As the first step let's generate the dataset.

```julia
using GlobalSensitivity, OrdinaryDiffEq, Flux, SciMLSensitivity, LinearAlgebra
using Optimization, OptimizationOptimisers, Distributions, Copulas, CairoMakie

u0 = Float32[2.0; 0.0]
datasize = 30
tspan = (0.0f0, 1.5f0)

function trueODEfunc(du, u, p, t)
    true_A = [-0.1 2.0; -2.0 -0.1]
    du .= ((u .^ 3)'true_A)'
end
t = range(tspan[1], tspan[2], length=datasize)
prob = ODEProblem(trueODEfunc, u0, tspan)
ode_data = Array(solve(prob, Tsit5(), saveat=t))
```

Now we will define our Neural Network for the dynamics of the system. We will use
a 2-layer neural network with 10 hidden units in the first layer and the second layer.
We will use the `Chain` function from `Flux` to define our NN. A detailed tutorial on
is available [here](https://docs.sciml.ai/SciMLSensitivity/stable/examples/neural_ode/neural_ode_flux/).

```julia
dudt2 = Flux.Chain(x -> x .^ 3,
    Flux.Dense(2, 10, tanh),
    Flux.Dense(10, 2))
p, re = Flux.destructure(dudt2) # use this p as the initial condition!
dudt(u, p, t) = re(p)(u) # need to restrcture for backprop!
prob = ODEProblem(dudt, u0, tspan)

θ = [u0; p] # the parameter vector to optimize

function predict_n_ode(θ)
    Array(solve(prob, Tsit5(), u0=θ[1:2], p=θ[3:end], saveat=t))
end

function loss_n_ode(θ)
    pred = predict_n_ode(θ)
    loss = sum(abs2, ode_data .- pred)
    loss
end

loss_n_ode(θ)

callback = function (state, l) #callback function to observe training
    display(l)
    return false
end

# Display the ODE with the initial parameter values.
callback(θ, loss_n_ode(θ))

# use Optimization.jl to solve the problem
adtype = Optimization.AutoZygote()

optf = Optimization.OptimizationFunction((p, _) -> loss_n_ode(p), adtype)
optprob = Optimization.OptimizationProblem(optf, θ)

result_neuralode = Optimization.solve(optprob,
    OptimizationOptimisers.Adam(0.05),
    callback=callback,
    maxiters=300)
```

Now we will use the Shapley method to understand the impact of each parameter on the
resultant of the cost function. We will use the `Shapley` function from `GlobalSensitivity`
to compute the so called Shapley Effects. We will first have to define some distributions
for the parameters. We will use the standard `Normal` distribution for all the parameters.

First let's assume no correlation between the parameters. Hence the covariance matrix
is passed as the identity matrix.

```julia
d = length(θ)
mu = zeros(Float32, d)
#covariance matrix for the copula
Covmat = Matrix(1.0f0*I, d, d)
#the marginal distributions for each parameter
marginals = [Normal(mu[i]) for i in 1:d]

copula = GaussianCopula(Covmat)
input_distribution = SklarDist(copula, marginals)

function batched_loss_n_ode(θ)
    prob_func(prob,i,repeat) = remake(prob;u0 =θ[1:2,i], p=θ[3:end,i])
    ensemble_prob = EnsembleProblem(prob,prob_func=prob_func)
    sol = solve(ensemble_prob,Tsit5(),EnsembleThreads();saveat=t,trajectories=size(θ,2))
    out = zeros(size(θ,2))
    for i in 1:size(θ,2)
      out[i] = sum(abs2, ode_data .- sol[i])
    end
    return out
end

shapley_effects = gsa(batched_loss_n_ode, Shapley(;n_perms = 100, n_var = 100, n_outer = 10), input_distribution, batch = true)
```

```julia
[ Info: Since `n_perms` wasn't set the exact version of Shapley will be used
GlobalSensitivity.ShapleyResult{Vector{Float64}, Vector{Float64}}([0.11597691741361442, 0.10537266345858425, -0.011525418832504125, 0.019080490852392638, 0.0670556101993216, -0.0008750631360554604, -0.06145135053766362, 0.04681820267596843, -0.052194422120816236, 0.003179470815545183  …  0.017116063071811214, -0.01996361592991698, 0.04992156377132031, -0.026031285685145327, -0.05478798810114203, -0.08297800907245817, -0.0007407741548139723, -0.004732287469108539, 0.0387866269216672, 0.003387278477023375], [0.24632058593157738, 0.2655832791843761, 0.2501184872448763, 0.24778944872968212, 0.24095751488197525, 0.25542101332993983, 0.23147124018855053, 0.2559000952299014, 0.2445090237211431, 0.24422366968866593  …  0.26334364579925296, 0.2533883745742706, 0.278493369461382, 0.251080668076158, 0.2513237168712358, 0.2565579384455956, 0.24086196097600907, 0.23509270698557308, 0.2424725703788857, 0.245598352488518], [-0.3668114310122772, -0.4151705637427929, -0.5017576538324616, -0.4665868286577843, -0.40522111896934987, -0.5015002492627375, -0.5151349813072227, -0.45474598397463833, -0.5314321086142567, -0.47549892177424  …  -0.4990374826947246, -0.5166048300954874, -0.49592544037298836, -0.5181493951144149, -0.5473824731687641, -0.5858315684258255, -0.47283021766779176, -0.46551399316083175, -0.43645961102094877, -0.4779854924004719], [0.5987652658395061, 0.6259158906599613, 0.47870681616745336, 0.5047478103625695, 0.5393323393679931, 0.4997501229906266, 0.39223228023189544, 0.5483823893265752, 0.4270432643726243, 0.48185786340533043  …  0.5332696088383471, 0.4766775982356534, 0.5957685679156289, 0.4660868237441243, 0.43780649696648005, 0.4198755502809092, 0.4713486693581638, 0.4560494182226147, 0.5140328648642831, 0.4847600493545186])
```

```julia
fig = Figure(resolution = (600, 400))
ax = barplot(fig[1,1], collect(1:54), shapley_effects.shapley_effects, color = :green)
ylims!(ax.axis, 0.0, 0.2)
ax.axis.xticks = (1:54, ["θ$i" for i in 1:54])
ax.axis.ylabel = "Shapley Indices"
ax.axis.xlabel = "Parameters"
ax.axis.xticklabelrotation = 1
display(fig)
```

![shapnocorr](https://github.com/SciML/GlobalSensitivity.jl/assets/23134958/d102a91a-a4ed-4850-ae0b-dea19acf38f7)

Now let's assume some correlation between the parameters. We will use a correlation of 0.09 between
all the parameters.

```julia
Corrmat = fill(0.09f0, d, d)
for i in 1:d
    Corrmat[i, i] = 1.0f0
end

#since the marginals are standard normal the covariance matrix and correlation matrix are the same
copula = GaussianCopula(Corrmat)
input_distribution = SklarDist(copula, marginals)
shapley_effects = gsa(batched_loss_n_ode, Shapley(;n_perms = 100, n_var = 100, n_outer = 100), input_distribution, batch = true)
```

```julia
[ Info: Since `n_perms` was set the random version of Shapley will be used
GlobalSensitivity.ShapleyResult{Vector{Float64}, Vector{Float64}}([0.05840668971922802, 0.11052100820850451, 0.04371662911807708, 0.004023059190511713, 0.062410433686220866, 0.02585247272606055, 0.02522310040104824, -0.002943508605003048, 0.0274450019985079, -0.00470104493101865  …  0.03521661871178529, -0.0029363975954207434, -0.01733340691251318, 0.030698550673315273, 0.004089097508271804, 0.01120562788725685, 0.020296678638214393, 0.05371236372397007, 0.041498171895387174, -0.0013082313559600266], [0.17059528049109549, 0.17289231112530734, 0.15914829962624458, 0.1353652701868229, 0.16197334249699738, 0.14706777933327114, 0.13542553650272246, 0.13442651137664902, 0.1187565412647469, 0.14591351295876914  …  0.14961042326057278, 0.11239722253111033, 0.14973761270843763, 0.15863981406666988, 0.14738095559109785, 0.11703801097937833, 0.1573849698258368, 0.16993657676693683, 0.1501327274957186, 0.1755876094816281], [-0.27596006004331913, -0.22834792159709788, -0.2682140381493623, -0.2612928703756612, -0.255057317607894, -0.2624003747671509, -0.24021095114428775, -0.2664194709032351, -0.205317818880396, -0.29069153033020617  …  -0.2580198108789374, -0.223234953756397, -0.3108191278210509, -0.28023548489735767, -0.28477757545028, -0.21818887363232467, -0.28817786222042574, -0.2793633267392261, -0.25276197399622125, -0.3454599459399511], [0.3927734394817752, 0.4493899380141069, 0.3556472963855164, 0.2693389887566846, 0.3798781849803357, 0.314105320219272, 0.29065715194638425, 0.260532453693229, 0.2602078228774118, 0.2812894404681689  …  0.32845304830250793, 0.2173621585655555, 0.2761523139960246, 0.34163258624398823, 0.2929557704668236, 0.24060012940683836, 0.3287712194968545, 0.3867880541871663, 0.3357583177869956, 0.34284348322803104])
```

```julia
fig = Figure(resolution = (600, 400))
ax = barplot(fig[1,1], collect(1:54), shapley_effects.shapley_effects, color = :green)
ylims!(ax.axis, 0.0, 0.2)
ax.axis.xticks = (1:54, ["θ$i" for i in 1:54])
ax.axis.ylabel = "Shapley Indices"
ax.axis.xlabel = "Parameters"
ax.axis.xticklabelrotation = 1
display(fig)
```

![shapcorr](https://github.com/SciML/GlobalSensitivity.jl/assets/23134958/c48be7e3-811a-49de-8388-4af1e03d0663)
