# Super MARIO: process Supervision without process for MAth Reasoning with code Interpreter Output

- [x] [Greedy](https://github.com/MARIO-Math-Reasoning/Super_MARIO?tab=readme-ov-file#greedy-decoding)
- [x] [Step Beam](https://github.com/MARIO-Math-Reasoning/Super_MARIO?tab=readme-ov-file#step-level-beam-search)
- [x] [MCTS](https://github.com/MARIO-Math-Reasoning/Super_MARIO?tab=readme-ov-file#mcts)

Download checkpoint [AlphaMath-7B 🤗](https://huggingface.co/MARIO-Math-Reasoning/AlaphaMath-7B)

This is the official repository for paper [AlphaMath Almost Zero: process Supervision without process](https://arxiv.org/abs/2405.03553). It is extracted from our internal corporate codebase. As a result, there may be slight differences when reproducing the numbers reported in our paper, but they should be very close. Our approach involves training the policy and value models using only the mathematical reasoning derived from the Monte Carlo Tree Search (MCTS) framework, eliminating the need for GPT-4 or human annotations. This is an illustration of [training instance](imgs/mcts_example.pdf) generated by MCTS in round 3.

## Inference on MATH dataset

| Inference Method       | Accuracy | avg. time (s) per q | avg. steps | # sols | 
| ---------------------- |:--------:|:-------------------:|:----------:|:------:|
| Greedy                 | 53.62    | 1.6                 | 3.1        | 1      |
| Maj@5                  | 61.84    | 2.9                 | 2.9        | 5      |  
| Step-level Beam (1,5)  | 62.32    | 3.1                 | 3.0        | 1      |
| &nbsp;&nbsp; + Maj@5   | 67.04    | &nbsp;&nbsp; x5     | &nbsp;&nbsp; x1  | 5|
| Step-level Beam (2,5)  | 64.66    | 2.4                 | 2.4        | 1      |
| Step-level Beam (3,5)  | 65.74    | 2.3                 | 2.2        | 1      |
| Step-level Beam (5,5)  | 65.98    | 4.7                 | 2.3        | 1      |
| &nbsp;&nbsp; + Maj@5   | 66.54    | &nbsp;&nbsp; x1     | &nbsp;&nbsp; x1  | 5|
| MCTS (N=40)            | 64.02    | 10.1                | 3.8        | 1      |


## Installation
1. Install `requirements.txt`
```
pip install -r requirements.txt
```
2. Install the [evaluation toolkit](https://github.com/MARIO-Math-Reasoning/MARIO_EVAL?tab=readme-ov-file#install-as-python-package) as a package.
3. Install our customized [vllm](https://github.com/MARIO-Math-Reasoning/vllm) to support value model.


## Checkpoint Initialization
1. Download the [deepseek-math-7b-base](https://huggingface.co/deepseek-ai/deepseek-math-7b-base).
2. You can use the `scripts/save_value_head.py` to add the value head to the LLM.


## Greedy Decoding
You can run either of the following two cmds. There may be slightly difference of accuracy between the two. In our machine, the first got 53.4% and the second got 53.62%.
```
python react_batch_demo.py \
--custom_cfg configs/react_sft.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```
or
```
# use step_beam (1, 1) without value func
python solver_demo.py \
--custom_cfg configs/sbs_greedy.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```


## Step-level Beam Search
In our machine, on MATH testset, the following cmd with config `B1=1, B2=5` can achieve ~62%, and the one with config `B1=3, B2=5` can reach ~65%.
```
python solver_demo.py \
--custom_cfg configs/sbs_sft.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```


## MCTS
### Training data generation. 

![](https://github.com/MARIO-Math-Reasoning/Super_MARIO/blob/main/imgs/mcts.png)

The `ground_truth` (the final answer, not the solution process) must be provided in `qaf` file.

round 1
```
# Checkpoint Initialization is required by adding value head
python solver_demo.py \
--custom_cfg configs/mcts_round1.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```

round > 1, after SFT
```
python solver_demo.py \
--custom_cfg configs/mcts_sft_round.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```

### Inference. 

Only `question` will be used, but the `ground_truth` will be used for calculating the accuracy..
```
python solver_demo.py \
--custom_cfg configs/mcts_sft.yaml \
--qaf ../MARIO_EVAL/data/math_testset_annotation.json
```
Different from step-level beam search, you need first to build a complete tree, then you should run the MCTS offline.
```
python offline_inference.py \
--custom_cfg configs/offline_inference.yaml \
--tree_jsonl <the saved tree jsonl file by solver_demo.py>
```
Note: this script can also be run with saved tree by step-level beam search, and the accuracy should remain the same.



## Value Estimation

### Distribution of Q-value for intermediate steps on training data. 

Because ground truth is known for training data, the value of final step is reward and Q-value can converge very well.

<img src="imgs/Q_distribution.png" width="500">

### Distribution of Q-value for both intermediate and final steps on test data. 

On test set, the ground truth is unknown, so the Q-value distribution includes both intermediate and final steps. From this figure, we can find
1. When model prediction is correct, its Q-value also converges towards 1.
2. For solutions with incorrect final answer, the distribution of Q-value covers all [-1,1], because the intermediate steps may be correct.
3. When the value model believes the solution predicted by the policy model to be incorrect, the Q-values cluster around $-1$.
4. There are instances where the value model erroneously considers an incorrect solution as correct, where the Q-values have a peak rouand 1. This pattern represents the bad cases of the value model.

<img src="imgs/Q_distribution_test.png" width="500">


## Citation
MCTS version
```
@misc{chen2024alphamath,
      title={AlphaMath Almost Zero: process Supervision without process}, 
      author={Guoxin Chen and Minpeng Liao and Chengxi Li and Kai Fan},
      year={2024},
      eprint={2405.03553},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```

Evaluation toolkit
```
@misc{zhang2024mario,
      title={MARIO Eval: Evaluate Your Math LLM with your Math LLM--A mathematical dataset evaluation toolkit}, 
      author={Boning Zhang and Chengxi Li and Kai Fan},
      year={2024},
      eprint={2404.13925},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```

OVM (Outcome Value Model) version
```
@misc{liao2024mario,
      title={MARIO: MAth Reasoning with code Interpreter Output -- A Reproducible Pipeline}, 
      author={Minpeng Liao and Wei Luo and Chengxi Li and Jing Wu and Kai Fan},
      year={2024},
      eprint={2401.08190},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```
