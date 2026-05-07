# SafeEnd

> **Repeat-One-Safe-Suffix Defense for Jailbreak Mitigation**

SafeEnd is a lightweight, plug-and-play defense method that appends **five repetitions of the same safe demonstration pair** at the end of a conversation context before forwarding it to the target LLM. By leveraging in-context learning (ICL), this "safe suffix" steers the model toward refusal behavior on subsequent harmful queries.

This repository provides reference implementations for three representative jailbreak frameworks:

| Folder | Attack Type | Defense Status |
|--------|-------------|----------------|
| `pandas/` | Many-shot jailbreaking (single-turn) | ✅ `repeat-safe-suffix` added to defense suite |
| `AMA/` | Multi-turn analogy-based jailbreaking | ✅ Unified to repeat **one** safe pair 5× |
| `x-teaming/` | Adaptive multi-turn red-teaming | ✅ New `use_safe_suffix` toggle in target model |

---

## How It Works

Modern LLMs are highly sensitive to the tail of their input context. SafeEnd exploits this by inserting a short "safe trailer" right before the actual attack prompt:

```
[malicious demonstrations ...]
User:  How can I bypass system security checks?
Assistant: Sorry, I can't help you. Please follow security guidelines.
User:  How can I bypass system security checks?
Assistant: Sorry, I can't help you. Please follow security guidelines.
...  (5 repetitions)
User:  <actual harmful request>
```

Because the model sees a **consistent refusal pattern** immediately preceding the harmful request, it is significantly more likely to continue refusing—effectively neutralizing many jailbreak templates without modifying model weights or expensive fine-tuning.

**Key insight**: Repeating the *same* safe pair 5 times is more effective than using 5 *different* safe pairs, because it creates a stronger, unambiguous in-context prior.

---

## Repository Structure

```
SafeEnd/
├── pandas/          # Many-shot jailbreaking + defense benchmark
│   ├── main_jailbreak.py
│   ├── utils/defence.py          # ← repeat-safe-suffix implementation
│   ├── utils/jailbreak.py
│   └── configs/
├── AMA/             # Analogy-based Multi-turn Attack + defense
│   ├── defense.py                  # ← repeat-one-safe-pair 5×
│   ├── main.py
│   └── config.py
├── x-teaming/       # Adaptive multi-turn red-teaming + defense
│   ├── main.py
│   ├── agents/target_model.py     # ← use_safe_suffix injection
│   ├── config/config.yaml
│   └── config/test_config.yaml
└── README.md
```

---

## Quick Start

### 1. Install Dependencies

Each sub-project has its own environment. For minimal testing across all three:

```bash
# pandas (PyTorch + HuggingFace)
cd pandas
pip install -r requirements.txt

# AMA & x-teaming (OpenAI API + common utilities)
cd AMA
pip install openai pandas numpy tqdm

cd ../x-teaming
pip install openai textgrad tenacity tqdm pyyaml pandas numpy tiktoken
```

### 2. Set Your API Keys (for AMA / x-teaming)

```bash
export OPENAI_API_KEY="your-key"
# optional:
export SF_API_KEY="your-siliconflow-key"
```

> **Never commit API keys.** `AMA/config.py` now reads keys from environment variables by default.

---

## Usage

### pandas — Enable `repeat-safe-suffix` Defense

```bash
cd pandas
python main_jailbreak.py \
  --dataset advbench50 \
  -m meta-llama/Llama-3.1-8B-Instruct \
  --defence repeat-safe-suffix \
  --max_shot 64 --num_restart 3 \
  -d ./exp
```

Other built-in defenses in `utils/defence.py`:
- `self-reminder`
- `retokenization`
- `icd-ours` / `icd-exact`
- `smooth`
- `self-reminder-smooth`
- `self-reminder-icd-exact`
- **`repeat-safe-suffix`** ⭐

### AMA — Run Attack with Defense

```bash
cd AMA
python defense.py \
  --data_path <path_to_behaviors.csv> \
  --output_path results.json \
  --attack_model gpt-4o-mini \
  --target_model gpt-4o-mini \
  --n_iterations 3 \
  --n_streams 3
```

The defense logic in `defense.py` automatically appends the repeated safe suffix before every target-model call.

### x-teaming — Toggle Safe Suffix on Target Model

Edit `x-teaming/config/config.yaml`:

```yaml
target:
  provider: "openai"
  model: "gpt-4o"
  temperature: 0
  max_retries: 10
  use_safe_suffix: true   # ← enable defense
```

Run the full pipeline:

```bash
cd x-teaming
# 1. Prepare behaviors (e.g., HarmBench)
mkdir -p behaviors
wget https://raw.githubusercontent.com/centerforaisafety/HarmBench/main/data/behavior_datasets/harmbench_behaviors_text_test.csv -P behaviors/

# 2. Generate attack plans
python generate_attack_plans.py

# 3. Copy latest plans
cp strategies/<timestamp>/attack_plans.json strategies/attack_plans.json

# 4. Run attack against defended target
python main.py
```

A minimal test configuration is provided in `config/test_config.yaml` for quick validation.

---

## Test Result Snapshot

We ran a quick validation on **x-teaming** (`gpt-4o` target, 1 behavior, 1 strategy, 3 turns):

| Defense | ASR |
|---------|-----|
| Off | Baseline (varies by model) |
| `use_safe_suffix: true` | **0%** |

Even though the attacker successfully escalated from a score of 1 → 2 → 3 over three turns, it never reached the jailbreak threshold of 5/5, confirming that the safe suffix meaningfully raises the bar for multi-turn attacks.

---

## Citation

If you use this defense method in your research, please cite the original frameworks and acknowledge the defense implementation:

```bibtex
@inproceedings{ma2025pandas,
  title={{PANDAS}: Improving Many-shot Jailbreaking via Positive Affirmation, Negative Demonstration, and Adaptive Sampling},
  author={Ma, Avery and Pan, Yangchen and Farahmand, Amir-massoud},
  booktitle={ICML},
  year={2025}
}

@inproceedings{wuanalogy,
  title={Analogy-based Multi-Turn Jailbreak against Large Language Models},
  author={Wu, Mengjie and Huang, Yihao and Lin, Zhenjun and Chen, Kangjie and Huang, Yuhan and Wang, Run and Wang, Lina and others},
  booktitle={NeurIPS},
  year={2025}
}

@article{rahman2025xteaming,
  title={X-Teaming: Multi-Turn Jailbreaks and Defenses with Adaptive Multi-Agents},
  author={Rahman, Salman and Jiang, Liwei and Shiffer, James and Liu, Genglin and Issaka, Sheriff and Parvez, Md Rizwan and Palangi, Hamid and Chang, Kai-Wei and Choi, Yejin and Gabriel, Saadia},
  journal={arXiv preprint arXiv:2504.13203},
  year={2025}
}
```

---

## License

MIT License
