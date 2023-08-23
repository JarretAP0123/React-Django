# React-Django


from flask import Flask, request, jsonify, Response
import torch
import transformers
from transformers import AutoTokenizer
from flask_cors import CORS
from functools import wraps
import secrets

app = Flask(__name__)
CORS(app)

# Backend 1 setup
model_name_1 = "meta-llama/Llama-2-7b-chat-hf"
tokenizer_1 = AutoTokenizer.from_pretrained(model_name_1)
model_1 = transformers.AutoModelForCausalLM.from_pretrained(model_name_1)
device_1 = torch.device("cuda:1")
model_1.to(device_1)

# Backend 2 setup
model_name_2 = "georgesung/llama2_7b_chat_uncensored"
tokenizer_2 = AutoTokenizer.from_pretrained(model_name_2)
model_2 = transformers.AutoModelForCausalLM.from_pretrained(model_name_2)
if torch.cuda.device_count() > 1:
    model_2 = torch.nn.DataParallel(model_2)
device_2 = torch.device("cuda:0")
model_2.to(device_2)

api_keys = {}

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        key = request.headers.get('x-api-key')
        if key not in api_keys:
            return Response('Unauthorized', 401)
        return f(*args, **kwargs)
    return decorated_function

@app.route('/generate_key', methods=['POST'])
def generate_api_key():
    key = secrets.token_urlsafe(32)
    api_keys[key] = True
    return jsonify({"api_key": key})

@app.route('/generate', methods=['POST'])
@require_api_key
def generate_text_backend_1():
    content = request.json
    prompt = content.get('prompt', '')
    inputs = tokenizer_1.encode(prompt, return_tensors="pt", add_special_tokens=True)
    inputs = inputs.to(device_1)
    with torch.no_grad():
        output = model_1.generate(
            inputs,
            do_sample=True,
            top_k=10,
            num_return_sequences=1,
            eos_token_id=tokenizer_1.eos_token_id,
            max_length=1024
        )
    generated_text = tokenizer_1.decode(output[0], skip_special_tokens=True)
    return jsonify({"generated_text": generated_text})

@app.route('/generate2', methods=['POST'])
def generate_text_backend_2():
    content = request.json
    prompt = content.get('prompt', '')
    inputs = tokenizer_2.encode(prompt, return_tensors="pt", add_special_tokens=True)
    inputs = inputs.to(device_2)
    with torch.no_grad():
        output = model_2.module.generate(
            inputs, 
            do_sample=True,
            top_k=10,
            num_return_sequences=1,
            eos_token_id=tokenizer_2.eos_token_id,
            max_length=4096
        )
    generated_text = tokenizer_2.decode(output[0], skip_special_tokens=True)
    return jsonify({"generated_text": generated_text})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)


