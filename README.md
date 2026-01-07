## Ground Truth Data and Question Generation

The ground truth data files should be in CSV format, consisting all neccessary keys for questions and corresponding answers. To form a prompt (or question), you must also prepare a JSON template, containing the components below:

- `task_name`: The name of the task
- `input_mode`: The mode of input, either choose `structured` to form questions using different question templates, or choose `qa_pairs` to directly feed in question-answer pairs.

If the `input_mode` is `structured`, you must also provide the following keys:

- `num_keys`: The number of keys
- `keys`: A list of keys that will be used to form questions. **Be sure that the keys are identical to the column names in the ground truth data file**
- `num_templates`: The number of templates
- `min_valid_answers_per_unit`: An integer threshold. A unit is considered covered/answerable only if the number of valid predictions (parsable as Final Answer: Yes/No) is at least this value. Only covered units are included in the accuracy (and consistency) denominators.  
- `question_templates`: A list of template strings, where placeholders are written as {key} and key must be in `keys`.
- `tie`: Decide how to mark the majority vote when the number of Yes and No among valid predictions are equal. Choose from "Yes", "No", or "Ambiguous". If `tie` = "Ambiguous", it is always treated as incorrect, and `ambiguous_rate` will be reported.