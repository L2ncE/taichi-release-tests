# taichi-release-tests

This repo contains scripts for running examples in taichi main repo, DiffTaichi, QuanTaichi, and more.
This is part of the testing process to ensure real world applications behave as expected.


## How to run

1. Dependencies

The only dependency is `PyYAML`:

```
pip install PyYAML
```

2. Clone / symlink relevant repos to the `repos` directory

```
ln -sf /path/to/taichi repos/taichi
```

3. Run examples with configured timelines

```
# Run
python3 run.py --log=DEBUG timelines/

# Run 3 instances simultaneously
python3 run.py --log=DEBUG --runners 3 timelines/

# Regenerate captures
python3 run.py --log=DEBUG --generate-captures timelines/
```


## Conventions

1. One or multiple yaml file per example in `timelines` directory, but not one yaml file for multiple examples.
2. Put repos being tested in `repos` directory`
3. Put captures in `truths` directory. 


## How to add your own test

Using the `waterwave.py` in taichi main repo as an example:

### Prerequisites

1. Activate a virtualenv with a usable taichi installation.
2. Ensure you have followed instruction 1 & 2 in `How to run` section.

### Generate interaction records

```bash
# Run waterwave.py, and write recorded timeline to waterwave.yaml
$ python record.py repos/taichi/python/taichi/examples/simulation/waterwave.py waterwave.yaml
... Click click ...

$ cat waterwave.yaml
```

```yaml
---
- path: repos/taichi/python/taichi/examples/simulation/waterwave.py
  args: []
  steps:
  - {frame: 1, action: move, position: [0.574, 0.568]}
  - {frame: 0, action: move, position: [0.572, 0.574]}
  - {frame: 0, action: mouse-down, key: LMB}
  - {frame: 0, action: move, position: [0.568, 0.592]}
  - {frame: 0, action: move, position: [0.566, 0.605]}
  - {frame: 0, action: mouse-up, key: LMB}
  - {frame: 0, action: move, position: [0.564, 0.609]}
  - {frame: 1, action: move, position: [0.424, 0.646]}
  - {frame: 0, action: mouse-down, key: LMB}
  - {frame: 0, action: move, position: [0.406, 0.646]}
  - {frame: 8, action: key-down, key: Super_L}
  - {frame: 30, action: succeed}
```

### Manually tune generated yaml

`record.py` can only record keyboard and mouse events, more advanced `steps` can be added manually.

#### Step format

````yaml
- frame: 30     # Common and mandatory arg, frame delay since last action
  action: move  # Common and mandatory arg, what to do when target frame is arrived.
  position:     # Specific arg for action `move`. In this example, 
  - 0.424
  - 0.646
````

#### Currently available actions

##### succeed & fail

````yaml
- frame: 30
  action: succeed
````

````yaml
- frame: 30
  action: fail
````

Self explanatory

##### (key|mouse)-(up|down) & key-press & mouse-click & move

These are keyboard & mouse events.
Generally you don't pay attention to them, these can be generated by `record.py` utility.


##### capture-and-compare

```yaml
- frame: 5
  action: capture-and-compare
  compare: sum-difference
  threshold: 1000
  ground_truth: truths/taichi/simulation/fractal.png
```


Captures image in GUI window, compare it to a specified ground truth.

`compare` can have these methods for now:

| compare             | Description                                                                      |
| ------------------- | -------------------------------------------------------------------------------- |
| sum-difference      | Calc `sum(abs(a[i] - b[i]) for i in <every-pixel>)`                              |
| blur-sum-difference | Apply gaussian blur to both capture and ground truth, calculate `sum-difference` |
| pixel-count         | Count every different pixel                                                      |
| rmse                | Calc `sqrt(sum((a[i] - b[i])**2 for i in <every-pixel>) / <total-pixel-count>)`  |

`threshold` can specify a percentage(as a string, like `"0.01%"`).
`ground_truth` is a path to png file, resides in `truths` directory.

Example: [fractal.yaml](timelines/taichi/simulation/fractal.yaml)


##### poke
```yaml
- frame: 30
  action: poke
  function: main
  code: |
    # Python code
```

At target (graphical) frame, run given python `code` in specified `function`'s (stack) frame.
Specified python code can freely access local & global variables in target `function`.

If you want to target module level frame, use `function: "<module>"`.

This is used for poking adjustable parameters of target program,
since this runner cannot interact with GUI components.

Example: [diff_sph.yaml](timelines/taichi/autodiff/diff_sph/diff_sph.yaml)


### Integration with `taichi` CI

For now `taichi` CI will run this test for every PR and master merge.

If you added a test for a different repo, you should also modify `taichi` CI script,
clone your target repo to `repos` directory.

[*nix Test Script](https://github.com/taichi-dev/taichi/blob/master/.github/workflows/scripts/unix_test.sh#L65)

[Windows Test Script](https://github.com/taichi-dev/taichi/blob/master/.github/workflows/scripts/win_test.ps1#L30)


### Known limitations

1. This runner only applies to python programs.
2. Simulated interactions cannot interact with GUI components (IMGUI), please use `poke` to change parameter value.
3. A lot of programs are not numerically stable, so you can't naively capture-and-compare. If you insist, try using fp64 instead of fp32.
