---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.1
kernelspec:
  display_name: Python 3.9.7 ('blackjax')
  language: python
  name: python3
mystnb:
  execution_timeout: 200
---

# Change of Variable in HMC


**Rat tumor problem:** We have J certain kinds of rat tumor diseases. For each kind of tumor, we test $N_{j}$ people/animals and among those $y_{j}$ tested positive. Here we assume that $y_{j}$ is distrubuted with **Binom**($N_{i}$, $\theta_{i}$). Our objective is to approximate $\theta_{j}$ for each type of tumor.

In particular we use following binomial hierarchical model where $y_{j}$ and $N_{j}$ are observed variables.


<table>
<tr>
</tr>
<tr>
<td>

![image.png](https://github.com/karm-patel/karm-patel.github.io/blob/master/images/binomial_hierarchichal.png?raw=True)

</td>
<td>

\begin{align}
    y_{j} &\sim \text{Binom}(N_{j}, \theta_{j}) \label{eq:1} \\
    \theta_{j} &\sim \text{Beta}(a, b) \label{eq:2} \\
    p(a, b) &\propto (a+b)^{-5/2}
\end{align}

</td>
</tr>
</table>

```{code-cell} ipython3
import arviz as az
import jax
import jax.numpy as jnp
import matplotlib.pyplot as plt
import pandas as pd
import seaborn as sns

pd.set_option("display.max_rows", 80)

import blackjax
import tensorflow_probability.substrates.jax as tfp

tfd = tfp.distributions
tfb = tfp.bijectors

plt.rc("xtick", labelsize=12)  # fontsize of the xtick labels
plt.rc("ytick", labelsize=12)  # fontsize of the tyick labels
```

## Data

```{code-cell} ipython3
# index of array is type of tumor and value shows number of total people tested.
group_size = jnp.array(
    [
        20,
        20,
        20,
        20,
        20,
        20,
        20,
        19,
        19,
        19,
        19,
        18,
        18,
        17,
        20,
        20,
        20,
        20,
        19,
        19,
        18,
        18,
        25,
        24,
        23,
        20,
        20,
        20,
        20,
        20,
        20,
        10,
        49,
        19,
        46,
        27,
        17,
        49,
        47,
        20,
        20,
        13,
        48,
        50,
        20,
        20,
        20,
        20,
        20,
        20,
        20,
        48,
        19,
        19,
        19,
        22,
        46,
        49,
        20,
        20,
        23,
        19,
        22,
        20,
        20,
        20,
        52,
        46,
        47,
        24,
        14,
    ],
    dtype=jnp.float32,
)

# index of array is type of tumor and value shows number of positve people.
n_of_positives = jnp.array(
    [
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        2,
        2,
        2,
        2,
        2,
        2,
        2,
        2,
        2,
        1,
        5,
        2,
        5,
        3,
        2,
        7,
        7,
        3,
        3,
        2,
        9,
        10,
        4,
        4,
        4,
        4,
        4,
        4,
        4,
        10,
        4,
        4,
        4,
        5,
        11,
        12,
        5,
        5,
        6,
        5,
        6,
        6,
        6,
        6,
        16,
        15,
        15,
        9,
        4,
    ],
    dtype=jnp.float32,
)

# number of different kind of rat tumors
n_rat_tumors = len(group_size)
```

```{code-cell} ipython3
plt.figure(figsize=(12, 3))
plt.bar(range(n_rat_tumors), n_of_positives)
plt.title("No. of positives for each tumor type", fontsize=14)
plt.xlabel("tumor type", fontsize=12)
sns.despine()
```

```{code-cell} ipython3
plt.figure(figsize=(14, 4))
plt.bar(range(n_rat_tumors), group_size)
plt.title("Group size for each tumor type", fontsize=14)
plt.xlabel("tumor type", fontsize=12)
sns.despine()
```

## Posterior Sampling

Now we use Blackjax's NUTS algorithm to get posterior samples of $a$, $b$, and $\theta$

```{code-cell} ipython3
from collections import namedtuple

params = namedtuple("model_params", ["a", "b", "thetas"])


def joint_logprob(params):
    # improper prior for a,b
    logprob_ab = jnp.log(jnp.power(params.a + params.b, -2.5))

    # logprob prior of theta
    logprob_thetas = tfd.Beta(params.a, params.b).log_prob(params.thetas).sum()

    # loglikelihood of y
    logprob_y = jnp.sum(
        tfd.Binomial(group_size, probs=params.thetas).log_prob(n_of_positives)
    )

    return logprob_ab + logprob_thetas + logprob_y
```

We take initial parameters from uniform distribution

```{code-cell} ipython3
rng_key = jax.random.PRNGKey(0)
n_params = n_rat_tumors + 2


def init_param_fn(seed):
    """
    initialize a, b & thetas
    """
    key1, key2, key3 = jax.random.split(seed, 3)
    return params(
        a=tfd.Uniform(0, 3).sample(seed=key1),
        b=tfd.Uniform(0, 3).sample(seed=key2),
        thetas=tfd.Uniform(0, 1).sample(n_rat_tumors, key3),
    )


init_param = init_param_fn(rng_key)
joint_logprob(init_param)  # sanity check
```

Now we use blackjax's window adaption algorithm to get NUTS kernel and initial states. Window adaption algorithm will automatically configure `inverse_mass_matrix` and `step size`

```{code-cell} ipython3
%%time
warmup = blackjax.window_adaptation(blackjax.nuts, joint_logprob, 1000)

# we use 4 chains for sampling
n_chains = 4
keys = jax.random.split(rng_key, n_chains)
init_params = jax.vmap(init_param_fn)(keys)

initial_states = jax.vmap(lambda seed, param: warmup.run(seed, param)[0])(
    keys, init_params
)

# can not vectorize kernel, since it is not jax.numpy array
_, kernel, _ = warmup.run(jax.random.PRNGKey(10), init_param_fn(rng_key))
```

Now we write inference loop for multiple chains

```{code-cell} ipython3
def inference_loop_multiple_chains(
    rng_key, kernel, initial_states, num_samples, num_chains
):
    @jax.jit
    def one_step(states, rng_key):
        keys = jax.random.split(rng_key, num_chains)
        states, infos = jax.vmap(kernel)(keys, states)
        return states, (states, infos)

    keys = jax.random.split(rng_key, num_samples)
    _, (states, infos) = jax.lax.scan(one_step, initial_states, keys)

    return (states, infos)
```

```{code-cell} ipython3
%%time
n_samples = 1000
states, infos = inference_loop_multiple_chains(
    rng_key, kernel, initial_states, n_samples, n_chains
)
```


## Arviz Plots

We have all our posterior samples stored in `states.position` dictionary and `infos` store additional information like acceptance probability, divergence, etc. Now, we can use certain diagnostics to judge if our MCMC samples are converged on stationary distribution. Some of widely diagnostics are trace plots, potential scale reduction factor (R hat), divergences, etc. `Arviz` library provides quicker ways to anaylze these diagnostics. We can use `arviz.summary()` and `arviz_plot_trace()`, but these functions take specific format (arviz's trace) as a input. So now first we will convert `states` and `infos` into `trace`.

```{code-cell} ipython3
def arviz_trace_from_states(states, info, burn_in=0):
    position = states.position
    if isinstance(position, jnp.DeviceArray):  # if states.position is array of samples
        position = dict(samples=position)
    else:
        try:
            position = position._asdict()
        except AttributeError:
            pass

    samples = {}
    for param in position.keys():
        ndims = len(position[param].shape)
        if ndims >= 2:
            samples[param] = jnp.swapaxes(position[param], 0, 1)[
                :, burn_in:
            ]  # swap n_samples and n_chains
            divergence = jnp.swapaxes(info.is_divergent[burn_in:], 0, 1)

        if ndims == 1:
            divergence = info.is_divergent
            samples[param] = position[param]

    trace_posterior = az.convert_to_inference_data(samples)
    trace_sample_stats = az.convert_to_inference_data(
        {"diverging": divergence}, group="sample_stats"
    )
    trace = az.concat(trace_posterior, trace_sample_stats)
    return trace
```

```{code-cell} ipython3
# make arviz trace from states
trace = arviz_trace_from_states(states, infos)
summ_df = az.summary(trace)
summ_df
```

**r_hat** is showing measure of each chain is converged to stationary distribution. **r_hat** should be less than or equal to 1.01, here we get r_hat far from 1.01 for each latent sample.

```{code-cell} ipython3
az.plot_trace(trace)
plt.tight_layout()
```

Trace plots also looks terrible and does not seems to be converged! Also, black band shows that every sample is diverged from original distribution. So **what's wrong happeing here?**

Well, it's related to support of latent variable. In HMC, the latent variable must be in an unconstrained space, but in above model `theta` is constrained in between 0 to 1. We can use change of variable trick to solve above problem

## Change of Variable
We can sample from logits which is in unconstrained space and in `joint_logprob()` we can convert logits to theta by suitable bijector (sigmoid). We calculate jacobian (first order derivaive) of bijector to tranform one probability distribution to another

```{code-cell} ipython3
transform_fn = jax.nn.sigmoid
log_jacobian_fn = lambda logit: jnp.log(jnp.abs(jnp.diag(jax.jacfwd(transform_fn)(logit))))
```

Alternatively, using the bijector class in `TFP` directly:

```{code-cell} ipython3
bij = tfb.Sigmoid()
transform_fn = bij.forward
log_jacobian_fn = bij.forward_log_det_jacobian
```

```{code-cell} ipython3
params = namedtuple("model_params", ["a", "b", "logits"])

def joint_logprob_change_of_var(params):
    # change of variable
    thetas = transform_fn(params.logits)
    log_det_jacob = jnp.sum(log_jacobian_fn(params.logits))

    # improper prior for a,b
    logprob_ab = jnp.log(jnp.power(params.a + params.b, -2.5))

    # logprob prior of theta
    logprob_thetas = tfd.Beta(params.a, params.b).log_prob(thetas).sum()

    # loglikelihood of y
    logprob_y = jnp.sum(
        tfd.Binomial(group_size, probs=thetas).log_prob(n_of_positives)
    )

    return logprob_ab + logprob_thetas + logprob_y + log_det_jacob
```

except change of variable in `joint_logprob()` function, everthing will remain same

```{code-cell} ipython3
rng_key = jax.random.PRNGKey(0)


def init_param_fn(seed):
    """
    initialize a, b & logits
    """
    key1, key2, key3 = jax.random.split(seed, 3)
    return params(
        a=tfd.Uniform(0, 3).sample(seed=key1),
        b=tfd.Uniform(0, 3).sample(seed=key2),
        logits=tfd.Uniform(-2, 2).sample(n_rat_tumors, key3),
    )


init_param = init_param_fn(rng_key)
joint_logprob_change_of_var(init_param)  # sanity check
```

```{code-cell} ipython3
%%time
warmup = blackjax.window_adaptation(blackjax.nuts, joint_logprob_change_of_var, 1000)

# we use 4 chains for sampling
n_chains = 4
keys = jax.random.split(rng_key, n_chains)
init_params = jax.vmap(init_param_fn)(keys)

initial_states = jax.vmap(lambda seed, param: warmup.run(seed, param)[0])(
    keys, init_params
)

# can not vectorize kernel, since it is not jax.numpy array
_, kernel, _ = warmup.run(jax.random.PRNGKey(10), init_param_fn(rng_key))
```

```{code-cell} ipython3
%%time
n_samples = 1000
states, infos = inference_loop_multiple_chains(
    rng_key, kernel, initial_states, n_samples, n_chains
)
```

```{code-cell} ipython3
# convert logits samples to theta samples
position = states.position._asdict()
position["thetas"] = jax.nn.sigmoid(position["logits"])
del position["logits"]  # delete logits
states = states._replace(position=position)
```

```{code-cell} ipython3
# make arviz trace from states
trace = arviz_trace_from_states(states, infos, burn_in=0)
summ_df = az.summary(trace)
summ_df
```

```{code-cell} ipython3
az.plot_trace(trace)
plt.tight_layout()
```

```{code-cell} ipython3
print(f"Number of divergence: {infos.is_divergent.sum()}")
```

We can see that **r_hat** is less than or equal to 1.01 for each latent variable, trace plots looks converged to stationary distribution, and only few samples are diverged.

+++

## Using a PPL

Probabilistic programming language usually provides functionality to apply change of variable easily (often done automatically). In this case for TFP, we can use its modeling API `tfd.JointDistribution*`.

```{code-cell} ipython3
tfed = tfp.experimental.distributions

@tfd.JointDistributionCoroutineAutoBatched
def model():
    # TFP does not have improper prior, use uninformative prior instead
    a = yield tfd.HalfCauchy(0, 100, name='a')
    b = yield tfd.HalfCauchy(0, 100, name='b')
    yield tfed.IncrementLogProb(jnp.log(jnp.power(a + b, -2.5)), name='logprob_ab')

    thetas = yield tfd.Sample(tfd.Beta(a, b), n_rat_tumors, name='thetas')
    yield tfd.Binomial(group_size, probs=thetas, name='y')

# Sample from the prior and prior predictive distributions. The result is a pytree.
# model.sample(seed=rng_key)
```

```{code-cell} ipython3
# Condition on the observed (and auxiliary variable).
pinned = model.experimental_pin(logprob_ab=(), y=n_of_positives)
# Get the default change of variable bijectors from the model
bijectors = pinned.experimental_default_event_space_bijector()

prior_sample = pinned.sample_unpinned(seed=rng_key)
# You can check the unbounded sample
# bijectors.inverse(prior_sample)
```

```{code-cell} ipython3
def joint_logprob(unbound_param):
    param = bijectors.forward(unbound_param)
    log_det_jacobian = bijectors.forward_log_det_jacobian(unbound_param)
    return pinned.unnormalized_log_prob(param) + log_det_jacobian
```

```{code-cell} ipython3
%%time
rng_key = jax.random.PRNGKey(0)
warmup = blackjax.window_adaptation(blackjax.nuts, joint_logprob, 1000)

# we use 4 chains for sampling
n_chains = 4
init_key, warmup_key = jax.random.split(rng_key, 2)
init_params = bijectors.inverse(pinned.sample_unpinned(n_chains, seed=init_key))

keys = jax.random.split(warmup_key, n_chains)
initial_states = jax.vmap(lambda seed, param: warmup.run(seed, param)[0])(
    keys, init_params
)

# can not vectorize kernel, since it is not jax.numpy array
_, kernel, _ = warmup.run(
    jax.random.PRNGKey(10),
    bijectors.inverse(pinned.sample_unpinned(seed=init_key)))
```

```{code-cell} ipython3
%%time
n_samples = 1000
states, infos = inference_loop_multiple_chains(
    rng_key, kernel, initial_states, n_samples, n_chains
)
```

```{code-cell} ipython3
# convert logits samples to theta samples
position = states.position
states = states._replace(position=bijectors.forward(position))
```

```{code-cell} ipython3
# make arviz trace from states
trace = arviz_trace_from_states(states, infos, burn_in=0)
summ_df = az.summary(trace)
summ_df
```

```{code-cell} ipython3
az.plot_trace(trace)
plt.tight_layout()
```
