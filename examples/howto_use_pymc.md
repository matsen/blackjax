---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.0
kernelspec:
  display_name: blackjax
  language: python
  name: python3
---

# Use with PyMC models

BlackJAX can take any log-probability function as long as it is compatible with JAX's primitives. In this notebook we show how we can use PyMC as a modeling language and BlackJAX as an inference library.

``` {admonition} Before you start
You will need [PyMC](https://github.com/pymc-devs/pymc) to run this example. Please follow the installation instructions on PyMC's repository.
```

We will reproduce the Eight School example from the [ TFP documentation](https://www.tensorflow.org/probability/examples/Eight_Schools). Follow the link for a description of the problem and the model that is used.

```{code-cell} ipython3
:tags: [hide-cell]
import numpy as np


J = 8
y = np.array([28.0, 8.0, -3.0, 7.0, -1.0, 1.0, 18.0, 12.0])
sigma = np.array([15.0, 10.0, 16.0, 11.0, 9.0, 11.0, 10.0, 18.0])
```

We implement the non-centered version of the hierarchical model:

```{code-cell} ipython3
import pymc as pm


with pm.Model() as model:

    mu = pm.Normal("mu", mu=0.0, sigma=10.0)
    tau = pm.HalfCauchy("tau", 5.0)

    theta = pm.Normal("theta", mu=0, sigma=1, shape=J)
    theta_1 = mu + tau * theta
    obs = pm.Normal("obs", mu=theta, sigma=sigma, shape=J, observed=y)
```


We need to translate the model into a log-probability function that will be used by Blackjax to perform inference. For that we use the `get_jaxified_logp` function in PyMC's internals.

```{code-cell} ipython3
from pymc.sampling_jax import get_jaxified_logp

rvs = [rv.name for rv in model.value_vars]
logprob_fn = get_jaxified_logp(model)
```

We can now run the window adaptation for the NUTS sampler:

```{code-cell} ipython3
import blackjax
import jax

# Get the initial position from PyMC
init_position_dict = model.initial_point()
init_position = [init_position_dict[rv] for rv in rvs]

rng_key = jax.random.PRNGKey(1234)

adapt = blackjax.window_adaptation(blackjax.nuts, logprob_fn, 1000)
last_state, kernel, _ = adapt.run(rng_key, init_position)
```

Let us now perform inference with the tuned kernel:

```{code-cell} ipython3
:tags: [hide-cell]

def inference_loop(rng_key, kernel, initial_state, num_samples):
    def one_step(state, rng_key):
        state, info = kernel(rng_key, state)
        return state, (state, info)

    keys = jax.random.split(rng_key, num_samples)
    _, (states, infos) = jax.lax.scan(one_step, initial_state, keys)

    return states, infos
```

```{code-cell} ipython3
states, infos = inference_loop(rng_key, kernel, last_state, 50_000)
```
