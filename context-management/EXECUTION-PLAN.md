# Fine-Tuning Execution Plan

## Quick Start

### Phase 1: Proof of Concept (Day 1)
1. Extract system prompt from claude-code
2. Generate 100 training examples
3. Fine-tune GPT-3.5-turbo ($5-10, 20 minutes)
4. Test behavior preservation
5. Measure token savings

### Fork Structure
```bash
claude-code-fork/
├── context-management/          # Planning and documentation
├── fine-tuning/                 # All fine-tuning work
│   ├── extract/
│   │   ├── extract_knowledge.py
│   │   └── knowledge_base.json
│   ├── generate/
│   │   ├── generate_dataset.py
│   │   └── validators.py
│   ├── datasets/
│   │   ├── training_v1.jsonl
│   │   └── validation.jsonl
│   ├── models/
│   │   └── model_registry.json
│   └── scripts/
│       ├── finetune.sh
│       └── test_model.py
└── tests/
    └── fine-tuning/
        ├── behavior_tests.py
        └── performance_tests.py
```

## Implementation Steps

### Step 1: Extract Knowledge
```python
# Extract system prompt, BMAD, rules, tool descriptions
python fine-tuning/extract/extract_knowledge.py
```

### Step 2: Generate Training Data
```python
# Convert knowledge to behavioral examples
python fine-tuning/generate/generate_dataset.py \
  --count 100 \
  --output datasets/test_v1.jsonl
```

### Step 3: Fine-Tune Model
```bash
# Quick test with GPT-3.5
python fine-tuning/scripts/finetune.py \
  --model gpt-3.5-turbo \
  --dataset datasets/test_v1.jsonl \
  --epochs 2
```

### Step 4: Configure Override
```javascript
// config/config.finetuned.json
{
  "api": {
    "url": "https://api.openai.com/v1/chat/completions",
    "model": "ft:gpt-3.5-turbo:personal::model-id",
    "system_prompt": "You are Claude-Code."  // Minimal
  }
}
```

### Step 5: Test & Validate
```bash
# Run A/B tests
CLAUDE_CODE_MODE=original pytest tests/fine-tuning/
CLAUDE_CODE_MODE=finetuned pytest tests/fine-tuning/
```

## Progressive Testing Phases

| Phase | Focus | Timeline | Examples Needed |
|-------|-------|----------|-----------------|
| 1 | System Prompt Only | 1 day | 100 |
| 2 | + BMAD Methodology | 2-3 days | 500 |
| 3 | + File Operations | 1 week | 2,000 |
| 4 | + Tool Descriptions | 2 weeks | 10,000 |
| 5 | Complete Knowledge | 1 month | 50,000+ |

## Expected Outcomes

### Token Savings
- **Current**: 40-60k context overhead
- **Phase 1**: 90% reduction
- **Phase 5**: 95-98% reduction

### Performance Impact
- **Before**: 60k system + 140k project = 200k (at limit)
- **After**: 0k system + 200k project = pure project focus

## Migration to Hive

Once validated in fork:
1. Package successful components
2. Create hive-integration module
3. Add to hive installer
4. Update configuration management

## Quick Validation Test

```python
# Generate 10 BMAD examples right now
examples = [
    "Create a REST API",
    "Debug memory leak",
    "Refactor component",
    "Add error handling",
    "Build auth system"
]

for task in examples:
    # Generate training example showing natural BMAD application
    # WITHOUT mentioning BMAD explicitly
    create_training_example(task)
```

## Success Metrics

- ✅ Behaviors preserved without system prompt
- ✅ 90%+ token reduction
- ✅ Consistent BMAD application
- ✅ Proper tool selection
- ✅ Error handling maintained

## Next Steps

1. Run extraction script on current system prompt
2. Generate initial 100 examples
3. Fine-tune test model ($5-10 investment)
4. Validate concept works
5. Scale up if successful