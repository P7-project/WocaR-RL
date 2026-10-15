# WocaR-PPO

This folder trains a simulated robot with PPO, a standard learning method. Two versions work:

- **Vanilla PPO.** Ordinary training. The robot is not taught to handle disturbed sensors.
- **WocaR-PPO.** The method from the paper. While it learns, it also practices against small changes to what the robot sees.

Choose the version by choosing a config file. A config is a JSON file of settings. Leave those files as they are.

Both versions train on Hopper and Walker. Vanilla PPO also trains on HalfCheetah.

## Install

Use Python 3.11. Open a terminal in `WocaR-PPO`, the folder that contains `requirements.txt`. If you are already in `src`, run `cd ..` first.

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cd src
```

On Windows PowerShell:

```powershell
py -3.11 -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
cd src
```

Stay in `src` with that environment active for every command below.

## Train

One command starts one new run, from the beginning. A full run is 976 updates, about 2 million steps of the robot, so it takes a while.

When it finishes, the last line says `Models saved to` and a path. The last folder in that path is the run id. Copy it. You need it to test.

Vanilla PPO:

```bash
python run.py --config-path config_hopper_vanilla_ppo.json --cpu true
python run.py --config-path config_walker_vanilla_ppo.json --cpu true
python run.py --config-path config_halfcheetah_vanilla_ppo.json --cpu true
```

WocaR-PPO:

```bash
python run.py --config-path config_hopper_robust_q_ppo_sgld.json --cpu true
python run.py --config-path config_walker_robust_q_ppo_sgld.json --cpu true
```

`--cpu true` runs without a GPU. `--cpu false` uses a GPU.

The run is saved next to the code, in a folder such as `vanilla_ppo_hopper/agents/<id>/` or `robust_q_ppo_sgld_hopper_best/agents/<id>/`. That folder holds the model (`store.h5`) and a `tensorboard/` log.

To see that a command starts, add `--force-stop-step 0`. It does one update, saves, and exits. The score will be low. Ctrl-C during a real run also saves what it has so far.

## Test

Use the same config you trained with. Replace the id below with the folder name from `Models saved to`. An id looks like `7c2dcd65-a381-4a63-8eb8-18474673415a`. If you leave the example in place, the test looks for that exact folder.

```bash
python test.py --config-path config_hopper_robust_q_ppo_sgld.json --cpu true --exp-id 7c2dcd65-a381-4a63-8eb8-18474673415a --deterministic
```

For Walker WocaR, change the config to `config_walker_robust_q_ppo_sgld.json` and use that run's id. For vanilla Hopper, use `config_hopper_vanilla_ppo.json`.

The command plays the robot with no window and prints a score for each game. The score is the reward. The length is how many steps it lasted before falling. Then it prints the average.

It plays 50 games. Pass `--num-episodes` only if you want a different number.

`--deterministic` turns off the small randomness in the robot's action, so the same situation always gets the same action and the score stays stable. Turn it on when you test. Leave it off when you train, because training needs that randomness, and `run.py` stops if you pass the flag.

## Attacks

An attack changes what the robot sees by a small amount, then you read the score again. Add it to the test command. The size should match training: Hopper `0.075`, Walker `0.05`, HalfCheetah `0.15`.

```bash
python test.py --config-path config_hopper_robust_q_ppo_sgld.json --cpu true --exp-id 7c2dcd65-a381-4a63-8eb8-18474673415a --deterministic --attack-eps 0.075 --attack-method random
```

Three attacks work with no extra files:

- `random` adds a random nudge inside that size.
- `action` searches for the nudge that changes the robot's action the most. The paper calls this MaxDiff.
- `critic` searches for the nudge that makes the value of the situation look worse.

`sarsa`, `advpolicy`, and `action+imit` are listed under [What still does not run](#what-still-does-not-run).

## Other flags

`--seed 1` on `run.py` makes the random numbers repeat. Leave it off to follow the authors, who set no seed.

`--row-id 0` on `test.py` tests an earlier save. The default, `-1`, tests the final model.

`--show-env true` can open a window only while a new training run creates the simulator. A saved run was stored without one, so the flag does nothing at test time.

## Leave the config files alone

The numbers in the JSON files are the authors' experiment: how long to train, the learning rates, how large a disturbance WocaR practices against, and WocaR's `q_weight` of 0.8. Changing them means you are running a different experiment.

A flag on the command line overrides the same setting in the JSON. That is how you set `--cpu` and the other flags above.

Vanilla files also contain robust-training fields, including `convex-relax`. Vanilla training skips those fields because `mode` is `ppo`. Leave them in the file.

## What was updated

The learning method is still the authors' method. These changes only make it run on a current machine:

- The old package pins (`torch` below 1.7, and `mujoco-py`) do not install anymore. This copy uses Python 3.11, PyTorch 2.2, Gymnasium, and MuJoCo 3.
- The robot names in the configs are now `Hopper-v4`, `Walker2d-v4`, `HalfCheetah-v4`, and `Ant-v4`.
- Hopper vanilla was given a few missing settings so the program would start. Vanilla training does not use them.

## What still does not run

Each of these was tried for one training update. They stop before a model is saved.

| Config | Why it stops |
| --- | --- |
| `config_walker_robust_ppo_convex.json` | Convex relaxation. PyTorch no longer has `_set_opset_version`. |
| `config_hopper_robust_ppo_convex.json` | Missing `value_activation`, `q_epochs`, `q_lr`, `tau`, `weight_schedule`, and `q_weight`. It quits before the convex code. |
| `config_ant_robust_ppo_convex.json` | Same missing settings as Hopper convex. |
| `config_halfcheetah_robust_q_ppo_sgld.json` | WocaR crashes. The memory network does not match the layer sizes. |
| `config_ant_robust_q_ppo.json` | Same crash as HalfCheetah WocaR. |
| `config_ant_vanilla_ppo.json` | Missing the same six settings. |
| `config_hopper_robust_ppo_sgld.json` | Missing `value_activation`. |
| `MuJoCo.json` | Missing the same six settings. |

`config_walker_robust_ppo_sgld.json`, `config_halfcheetah_robust_ppo_sgld.json`, and `config_halfcheetah_robust_ppo_convex.json` do train. The last one is SA-PPO, not convex relaxation.

These attacks stop because the extra network file is not in the folder. `random`, `action`, and `critic` ran.

| Attack | Why it stops |
| --- | --- |
| `sarsa` | Looks for `sarsa.model`. |
| `advpolicy` | Looks for `advpolicy.model`. |
| `action+imit` | Needs an imitation model in the same way. |