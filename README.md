from flask import Flask, request, jsonify
from flask_cors import CORS
from werkzeug.exceptions import HTTPException
from filters import is_safe
from config import Config
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
import logging

app = Flask(__name__)
app.config.from_object(Config)
CORS(app)

# Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("DivaAI")

# Rate Limiting
limiter = Limiter(get_remote_address, app=app, default_limits=[Config.RATE_LIMIT])

@app.errorhandler(Exception)
def handle_error(e):
    code = 500
    if isinstance(e, HTTPException):
        code = e.code
    logger.error(f"Error: {str(e)}")
    return jsonify({"success": False, "message": str(e)}), code

@app.route('/', methods=['GET'])
def home():
    return jsonify({"name": "Diva AI", "message": "Welcome to Diva AI!"})

def get_query_from_request():
    data = request.get_json(force=True)
    return str(data.get('query', ''))

@app.route('/respond', methods=['POST'])
@limiter.limit("30/minute")
def respond():
    query = get_query_from_request()
    if not is_safe(query):
        return jsonify({"success": False, "message": "Sorry, unsafe/forbidden content detected."}), 400
    return jsonify({"success": True, "response": f"Diva: I heard '{query}'. How can I help you more?"})

@app.route('/speak', methods=['POST'])
@limiter.limit("30/minute")
def speak():
    query = get_query_from_request()
    if not is_safe(query):
        return jsonify({"success": False, "message": "Sorry, unsafe/forbidden content detected."}), 400
    return jsonify({"success": True, "spoken": f"Speaking: {query}"})

@app.route('/search', methods=['POST'])
@limiter.limit("30/minute")
def search():
    query = get_query_from_request()
    if not is_safe(query):
        return jsonify({"success": False, "message": "Sorry, unsafe/forbidden content detected."}), 400
    return jsonify({"success": True, "search_results": [f"Result for '{query}'"]})

@app.route('/create_image', methods=['POST'])
@limiter.limit("10/minute")
def create_image():
    data = request.get_json(force=True)
    description = str(data.get('description', ''))
    if not is_safe(description):
        return jsonify({"success": False, "message": "Sorry, unsafe/forbidden content detected."}), 400
    return jsonify({"success": True, "image_url": "https://dummyimage.com/300.png/09f/fff?text=Diva+Image"})

if __name__ == '__main__':
    app.run(host=Config.HOST, port=Config.PORT)
