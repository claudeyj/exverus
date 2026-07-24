# Benchmarks

This directory holds the additional benchmarks used in the [ExVerus](../README.md) evaluation, beyond the original
150-task VerusBench set in [`cleaned-verusbench/`](../cleaned-verusbench/README.md). These four datasets correspond
to the harder/robustness benchmarks reported in the paper (Table 1, Table 2, `--source THREEBENCH`/`ADDITIONAL`).

## Overview

| Source | Directory | Tasks | Description | Citation |
|--------|-----------|-------|--------------|----------|
| DafnyBench (Dafny2Verus) | [`Dafnybench/`](Dafnybench) | 67 | Tasks from the DafnyBench dataset, translated from Dafny to Verus | Aggarwal et al., 2025 (translation); Loughridge et al., 2025 (original DafnyBench) |
| HumanEval-Verus | [`HumanEval_Alphaverus/`](HumanEval_Alphaverus) | 68 | Handwritten Verus translations of HumanEval, curated with an AlphaVerus-style filtering process | Bai et al., 2025 (translation); Chen et al., 2021 (original HumanEval) |
| LCBench (LeetCode-Verus) | [`Leetcode/`](Leetcode) | 28 | Challenging proof tasks manually translated from LeetCode by human experts (~200 LoC on average) | Dai, 2025 |
| ObfsBench | [`Obfuscated_Verusbench/`](Obfuscated_Verusbench) | 266 (267 verified) | VerusBench samples obfuscated under semantics-preserving transformations, for out-of-distribution robustness evaluation | Introduced by ExVerus (Appendix E) |

Each count matches the task count reported in the paper for that benchmark.

## Structure

Each benchmark folder follows the same layout as `cleaned-verusbench/`:

```
Benchmarks/
├── Dafnybench/                # DafnyBench -> Verus (67 tasks)
├── HumanEval_Alphaverus/       # HumanEval -> Verus (68 tasks)
├── Leetcode/                   # LCBench -> Verus (28 tasks)
└── Obfuscated_Verusbench/       # ObfsBench (266 tasks)
```

Each source folder contains:
- `unverified/` - Tasks without proof annotations to be solved
- `verified/` - The same tasks with correct proof annotations for reference

## Task Descriptions

- **DafnyBench**: 67 tasks translated from the original DafnyBench dataset (Loughridge et al., 2025) to Verus, following
  the Dafny2Verus pipeline (Aggarwal et al., 2025). To mitigate reward hacking from tautological specs, the source
  tasks were filtered with an LLM-as-judge majority vote over five prompt variations (Appendix G.2).
- **HumanEval-Verus**: 68 tasks translating the HumanEval benchmark (Chen et al., 2021) to Verus, curated using an
  approach similar to AlphaVerus (Bai et al., 2025).
- **LCBench (LeetCode-Verus)**: 28 tasks derived from the LeetCode platform, curated by human experts who manually
  translate LeetCode problems into Verus proofs; these are the most complex tasks in the suite, requiring extensive
  reasoning at ~200 LoC on average (Dai, 2025).
- **ObfsBench**: Introduced by this paper to test robustness against out-of-distribution inputs. Built by obfuscating
  both programs and proofs from VerusBench (Appendix E), covering three categories of semantics-preserving
  transformations:
  - **Layout**: e.g., identifier renaming (replacing descriptive names with generic ones).
  - **Data**: e.g., dead variable insertion, instruction substitution.
  - **Control flow**: e.g., dead code insertion, opaque predicates, control flow flattening.

  This yields 266 challenging yet verifiable out-of-distribution tasks.

## Citations

```bibtex
@inproceedings{aggarwal2025alphaverus,
  title={Alphaverus: Bootstrapping formally verified code generation through self-improving translation and treefinement},
  author={Aggarwal, P. and Parno, B. and Welleck, S.},
  booktitle={Forty-second International Conference on Machine Learning},
  year={2025},
  url={https://openreview.net/forum?id=tU8QKX4dMI}
}

@article{loughridge2025dafnybench,
  title={Dafnybench: A benchmark for formal software verification},
  author={Loughridge, C. R. and Sun, Q. and Ahrenbach, S. and Cassano, F. and Sun, C. and Sheng, Y. and Mudide, A. and Misu, M. R. H. and Amin, N. and Tegmark, M.},
  journal={Transactions on Machine Learning Research},
  year={2025},
  url={https://openreview.net/forum?id=yBgTVWccIx}
}

@misc{bai2025humanevalverus,
  title={Humaneval-verus: Handwritten examples of verified verus code derived from humaneval},
  author={Bai, A. and Bosamiya, J. and Fernando, E. and Hossain, M. R. and Lorch, J. and Lu, S. and Neamtu, N. and Parno, B. and Shah, A. and Tang, E.},
  year={2025},
  howpublished={\url{https://github.com/secure-foundations/human-eval-verus}}
}

@article{chen2021evaluating,
  title={Evaluating large language models trained on code},
  author={Chen, Mark and others},
  journal={CoRR},
  volume={abs/2107.03374},
  year={2021},
  url={https://arxiv.org/abs/2107.03374}
}

@misc{dai2025leetcodeverus,
  title={verus-study-cases-leetcode},
  author={Dai, W.},
  year={2025},
  howpublished={\url{https://github.com/WeituoDAI/verus-study-cases-leetcode}}
}
```

ObfsBench is introduced by the ExVerus paper itself; see the [Citation](../README.md#citation) section in the root
README.
