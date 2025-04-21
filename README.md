---
layout: post
title: "CAD-Assistance for Gen AI Intensive Course Capstone 2025Q1"
date: 2024-04-20 
---

## Introduction: Can AI Help Us Design Faster?

Learning 3D CAD (Computer-Aided Design) is essential for anyone wanting to bring physical ideas to life, especially for projects like custom robot parts or automotive modifications. While mastering the basics is achievable, translating a mental concept into the precise commands of a CAD tool and quickly iterating on prototypes can be a significant hurdle, especially for beginners.

This project explores using Generative AI, specifically Google's Gemini models, to bridge this gap. The goal was to create a "CAD Assistant" that takes natural language descriptions of mechanical parts and generates starter code for [CadQuery](https://cadquery.readthedocs.io/en/latest/), a powerful Python-based parametric CAD library. This aims to accelerate the prototyping workflow and provide a helpful starting point for users.

---

## The Approach: From Text to 3D Model

The core idea is straightforward:

1.  **User Input:** Provide a text prompt describing the desired part (e.g., "a 10x10x10 mm cube").
2.  **AI Processing:** Send this prompt, along with carefully crafted instructions and examples, to a Gemini LLM via the `google-generativeai` Python library.
3.  **Code Generation:** The LLM generates Python code using the CadQuery library syntax.
4.  **Execution & Visualization:** The generated code is executed, and if successful, the resulting 3D model is visualized directly within the Kaggle notebook using the `jupyter-cadquery` extension.

**Why CadQuery?** It's parametric (great for mechanical parts), uses Python (which LLMs are generally good at generating), and has excellent integration with Jupyter notebooks for visualization.

---

## Leveraging Gen AI Capabilities

This project progressively implemented several Gen AI capabilities to improve the assistant's robustness and functionality:

### 1. Few-Shot Prompting

Simply telling the LLM what to do (zero-shot) often yields poor results for specific formats like code. By providing examples of user requests and the expected CadQuery code output *within the prompt itself* (few-shot prompting), we significantly guide the model towards generating correct syntax.

```python
# Example structure shown to the LLM:
examples = """
User: Create a 10x10x10 mm cube.
Assistant:
import cadquery as cq
result = cq.Workplane("XY").box(10, 10, 10)

User: Make a solid cylinder with radius 5 and height 20.
Assistant:
import cadquery as cq
result = cq.Workplane("XY").cylinder(20, 5)
# ... more examples ...
"""
```
![image](https://github.com/user-attachments/assets/5935cabd-4ddd-4e61-9958-4fe9806b1f79)


### 2. Structured Output (JSON Mode)

Initially, the LLM might return code mixed with explanatory text or markdown formatting (like ```python ... ```). This makes reliably extracting *just* the code difficult. Instructing the model to return its output **strictly as a JSON object** with a defined key (e.g., `"code"`) makes parsing robust.

```python
# Modified prompt example asking for JSON:
examples = """
User: Create a 10x10x10 mm cube.
Assistant:
{"code":
"import cadquery as cq\\n\\
result = cq.Workplane('XY').box(10, 10, 10)"}
# ... more examples ...
"""

# Parsing the response in Python:
raw_extracted_code = json.loads(cleaned_json_text).get('code')
generated_code = raw_extracted_code.encode().decode('unicode_escape')
```
*(Note: Decoding escape sequences like `\\n` is necessary after JSON parsing).*

### 3. Gen AI Evaluation

How good is the generated code? We can ask another LLM! This capability involves sending the original user request and the generated code to the LLM again, prompting it to act as a code reviewer and return a structured evaluation (again using JSON for robustness).

```python
# Prompt structure for the evaluator LLM (requesting JSON output)
evaluation_instruction = """You are an expert Python and CadQuery code reviewer.
Analyze the provided user request and the generated CadQuery code.
Return your evaluation ONLY as a JSON object containing the following keys:
- "is_correct": String ("Yes", "No", "Partially")...
- "syntax_errors": String ("None" or description)...
- "issues_and_suggestions": List of strings...
- "improved_code": String or null...
- "assessment_summary": String...
Do NOT include any explanatory text before or after the JSON object.
"""
# ... (rest of evaluation prompt) ...

# Example evaluation output (as Python dict after parsing JSON):
# {'assessment_summary': 'The code is correct...',
#  'improved_code': None,
#  'is_correct': 'Yes',
#  'issues_and_suggestions': [],
#  'syntax_errors': 'None'}
```
This provides an automated first pass at quality control.

### 4. Controlled Generation (Parameters)

We can control the "creativity" vs. "predictability" of the LLM's text generation using parameters like `temperature`. Lower temperatures produce more focused, common outputs, while higher temperatures allow for more diverse, sometimes unexpected results. We demonstrated this by asking the LLM to creatively describe the generated CAD part with different temperature settings.

```python
# Calling the API with specific generation config
low_temp_config = types.GenerateContentConfig(temperature=0.2, ...)
high_temp_config = types.GenerateContentConfig(temperature=0.9, ...)

response_low_temp = client.models.generate_content(..., config=low_temp_config)
response_high_temp = client.models.generate_content(..., config=high_temp_config)

# Outputs showed more factual vs. more descriptive text respectively.
```

### 5. Document Understanding

Instead of just a short prompt, we can provide the LLM with a larger block of text (a "document") containing specifications and ask it to generate code based on that context.

```python
# Example spec document
gear_spec_document = """
Part Type: Simple Spur Gear
Material: PLA
Outer Diameter (mm): 40
Thickness (mm): 5
# ... more specs ...
"""

# Prompt referencing the document
doc_prompt = f"""{doc_instruction}

Specification Document Content:
---START DOCUMENT---
{document_content}
---END DOCUMENT---

Generated CadQuery Code:
"""
# The LLM then attempts to generate code using parameters from the document.
```
This allows for more complex input than a simple prompt string.

---

## Putting It All Together: The `cad_assistant.py` Script

To make the assistant reusable, the core logic incorporating these capabilities was consolidated into a Python script (`cad_assistant.py`). This script uses `argparse` to accept user input via command-line arguments:

*   `-p "prompt"`: Generate code from a natural language prompt.
*   `-d path/to/spec.txt`: Generate code based on a specification document.
*   `-t`, `--top_k`, `--top_p`: Control generation parameters.
*   `-e`: Enable the Gen AI evaluation step.
*   `-o filename.py`: Save the generated code to a file.

**Example Script Calls:**

```bash
# Box with hole (prompt + evaluation)
!python /kaggle/working/cad_assistant.py -p "Create a 30x30x10 mm box with a 15mm diameter hole..." -e -o /kaggle/working/generated_box_hole.py

# Cylinder with chamfer (prompt + parameters)
!python /kaggle/working/cad_assistant.py -p "Make a solid cylinder 50mm tall with radius 10mm. Add a 2mm chamfer..." -t 0.4 --top_p 0.9 -o /kaggle/working/generated_cylinder_chamfer.py

# Washer from document
!python /kaggle/working/cad_assistant.py -d /kaggle/working/washer_spec.txt -o /kaggle/working/generated_washer.py
```

---

## Results & Visualizations

The script successfully generated executable CadQuery code for the simpler geometric examples:

**1. Box with Centered Hole:**

```python
# Code generated by script...
import cadquery as cq
result = cq.Workplane('XY').box(30, 30, 10).faces('>Z').workplane().circle(7.5).cutThruAll()
```

![image](https://github.com/user-attachments/assets/f025ff5b-de40-4737-84ab-b443442dba0d)


**2. Cylinder with Chamfer:**

```python
# Code generated by script...
import cadquery as cq
result = cq.Workplane('XY').cylinder(50, 10).edges(">Z").chamfer(2)
```

![image](https://github.com/user-attachments/assets/d3eca7f6-4673-4338-ad13-bc13eac375f6)


**3. Thick Washer (from Document):**

```python
# Code generated by script...
import cadquery as cq
outer_diameter = 30
inner_diameter = 15
thickness = 4
result = cq.Workplane("XY").circle(outer_diameter / 2).circle(inner_diameter / 2).extrude(thickness)
```

![image](https://github.com/user-attachments/assets/c73743ae-6482-426d-b282-7d56efc29e80)

---

## Challenges and Limitations

While promising, this project also highlighted limitations:

*   **Complex Geometry:** Generating code for more intricate shapes like correctly joined L-brackets, precise involute gears (as seen in the Document Understanding attempt), or multi-part assemblies often failed. The LLM sometimes produced syntactically incorrect code (using outdated methods like `cq.occ` or wrong keyword arguments like `r` for `cylinder`) or code that didn't logically achieve the desired geometry.
*   **Prompt Sensitivity:** The quality of the generated code is highly dependent on the prompt's clarity, the quality of few-shot examples, and even minor changes in instructions (as seen when adding JSON output initially affected code correctness).
*   **Domain Knowledge Gap:** General LLMs lack deep, specialized knowledge of CAD principles and library nuances without specific fine-tuning or augmentation.

This assistant is currently best suited for generating simple base shapes or performing basic modifications, acting as a starting point rather than a complete CAD solution.

---

## Future Work

There are many exciting avenues for future development:

1.  **Image Understanding:** Integrate a multimodal model to interpret user sketches alongside text prompts (Sketch-to-Code).
2.  **RAG for Libraries:** Use Retrieval-Augmented Generation by creating a vector database of code examples from libraries like `cq-gears` or OpenSCAD modules. This would allow the LLM to generate code using specific, complex functions based on retrieved examples when prompted (e.g., "create a 12-tooth spur gear using cq-gears").
3.  **Function Calling:** Fully implement the function calling loop (using `genai.GenerativeModel`) to allow the LLM to use local tools (like syntax checkers or formatters) and potentially self-correct based on their output.
4.  **Interactive UI:** Build a web interface (Streamlit/Gradio) for easier use.
5.  **Error Feedback Loop:** Feed execution errors back to the LLM to request corrections.

---

## Links

*   **Kaggle Notebook:** https://www.kaggle.com/code/syedfarazhussaini/gen-ai-cad-assistant-prototyping-mechanical-parts

This exploration into Gen AI for CAD assistance shows exciting potential, particularly for simplifying initial design steps and aiding learning. As models and techniques evolve, tools like this could become increasingly powerful aids in the engineering and design process.

Below is the embedded Kaggle Notebook containing the full implementation and step-by-step demonstration:

<iframe src="https://www.kaggle.com/embed/syedfarazhussaini/gen-ai-cad-assistant-prototyping-mechanical-parts?kernelSessionId=235148734" height="800" style="margin: 0 auto; width: 100%;" frameborder="0" scrolling="auto" title="Gen AI CAD Assistant: Prototyping Mechanical Parts"></iframe>
