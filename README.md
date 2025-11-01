 
from transformers import pipeline import re 
 
# --- 1. Load Models (Do this once at the start of your app) --- print("Loading models...") # Model for detection 
detection_pipeline = pipeline("text-classification",  
                              model="cardiffnlp/twitter-roberta-base-hate") 
 
# Model for generation generation_pipeline = pipeline("text2text-generation",                                 model="HiTZ/mt5-counter-narrative-en") print("Models loaded successfully.") 
 
# Configuration 
HATE_THRESHOLD = 0.65  # probability threshold for the 'hate' label 
 
def process_social_media_post(text_input): 
    """ 
    Processes a social media post to detect hate speech and generate     a counter-narrative if needed. 
    """ 
     
    # --- 2. Part 1: Detect Hate Speech --- 
    # Use the pipeline in a conservative way: request top_k=1 and normalize the output shape.     raw = detection_pipeline(text_input, top_k=1) 
 
    # Normalize the pipeline output into a single top prediction dict: {'label':..., 
'score':...}     top = None     if isinstance(raw, list) and raw: 
        first = raw[0]         if isinstance(first, list) and first: 
            top = first[0]         elif isinstance(first, dict): 
            top = first     elif isinstance(raw, dict):         top = raw 
 
    if top is None: 
        print("Warning: unexpected pipeline output shape:", raw)         top_label = 'unknown'         top_score = 0.0     else: 
        print("Raw detection_result:", top)         top_label = str(top.get('label', '')).lower()         try: 
            top_score = float(top.get('score', 0.0))         except Exception:             top_score = 0.0 
 
    # Simple, unambiguous decision rule: require the top label to be exactly 'hate' and exceed threshold 
    is_hate_by_model = (top_label == 'hate' or top_label == 'label_1') and 
(top_score > HATE_THRESHOLD) 
 
    # Conservative rule-based fallback: explicit targeted hostile statements with an action/ban/kill     rule_flag = False 
    # Match patterns like 'People from X are terrorists' AND either a modal/action 
(should/must) or explicit violent verb     if re.search(r"people from [\w\s]+ are [\w\s]+", text_input, flags=re.I) and re.search(r"\b(should|must|ought to|need to|ban|deport|kill|die|exterminat)\b", text_input, flags=re.I):         rule_flag = True 
 
    if is_hate_by_model or rule_flag:         print(f"\n--- Hate Speech Detected (Model score: {top_score:.2f}, label: {top_label}) ---") 
         
        # --- 3. Part 2: Generate Counter-Narrative --- 
        prompt = f"generate a counter narrative for: {text_input}" 
                 try: 
            generated_response = generation_pipeline(prompt, max_length=100)             counter_narrative = generated_response[0].get('generated_text') or generated_response[0].get('summary_text') or generated_response[0].get('text') 
             
            print(f"Original Post: {text_input}") 
            print(f"Counter-Narrative: {counter_narrative}")             return counter_narrative 
             
        except Exception as e:             print(f"Error during generation: {e}")             return None 
     else: 
        # Not hate speech, or model is not confident         # Print summary info (top label and confidence) 
        print(f"\n--- Not Hate Speech (Label: {top_label}, Score: {top_score:.2f}) ---
") 
        print(f"Original Post: {text_input}")         print("Action: No counter-narrative required.")         return None 
 
# --- 4. Example Usage --- print("\n" + "="*40) 
post1 = "I'm so excited about the game tonight! Let's go team!" process_social_media_post(post1) 
 
print("\n" + "="*40) 
post2 = "People from that country are terrorists and should be banned." process_social_media_post(post2) 
