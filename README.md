# Flask Microservice Project

## Overview

This project demonstrates how to design and implement a microservice using Python. The microservice is designed to manage a product catalog for an eCommerce web application. Each component of the web application, including the product catalog, order list, and payment processing system, is implemented as an independent service. Services communicate with each other efficiently using HTTP.

### Prerequisites

- Familiarity with Flask
- Python, Postman Client, and Docker Desktop installed on your computer

## Project Setup

### Step 1: Create Your Project

1. Create a project directory named `flask-microservice` and navigate to it.
2. Confirm Python installation by running:
   ```bash
   python3 --version
   ```
3. Install `virtualenv` to create an isolated development environment:
   ```bash
   pip3 install virtualenv
   ```
4. Create and activate a virtual environment:
   ```bash
   virtualenv venv
   ```
   - On Windows:
     ```bash
      Set-ExecutionPolicy Unrestricted -Scope Process
     .\venv\Scripts\activate
     ```
     if
   - On Unix or macOS:
     ```bash
     source venv/bin/activate
     ```

### Step 2: Set Up a Flask Server

1. In the root directory, create a `requirements.txt` file and add these dependencies:
   ```
   flask
   requests
   ```
2. Install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Create a `services` folder in the root directory. Inside `services`, create a file named `products.py` with the following code:

   ```python
   import requests
   import os
   from flask import Flask, jsonify

   app = Flask(__name__)
   port = int(os.environ.get('PORT', 5000))

   @app.route("/")
   def home():
       return "Hello, this is a Flask Microservice"

   if __name__ == "__main__":
       app.run(debug=True, host="0.0.0.0", port=port)
   ```

### Step 3: Define API Endpoints

1. Update `products.py` to include an API endpoint for fetching product data:

   ```python
   BASE_URL = "https://dummyjson.com"

   @app.route('/products', methods=['GET'])
   def get_products():
       response = requests.get(f"{BASE_URL}/products")
       if response.status_code != 200:
           return jsonify({'error': response.json()['message']}), response.status_code
       products = []
       for product in response.json()['products']:
           product_data = {
               'id': product['id'],
               'title': product['title'],
               #'brand': product['brand'],
               'price': product['price'],
               'description': product['description']
           }
           products.append(product_data)
       return jsonify({'data': products}), 200 if products else 204
   ```

### Step 4: Test the Microservice

1. Start the development server:
   ```bash
   flask --app services/products run
   ```
2. Use Postman to send a GET request to `http://localhost:5000/products` and verify the response.

## Implementing Authentication and Authorization

1. Add `pyjwt` to your `requirements.txt` and reinstall dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Create a `users.json` file in the root directory:
   ```json
   [
     {
       "id": 1,
       "username": "admin",
       "password": "admin"
     }
   ]
   ```
3. Update `products.py` with necessary imports and JWT configurations:

   ```python
   import requests
   from flask import Flask, jsonify, request, make_response
   import jwt
   from functools import wraps
   import json
   import os
   from jwt.exceptions import DecodeError

   app.config['SECRET_KEY'] = os.urandom(24)
   ```

4. Create a decorator function for token verification:

   ```python
   def token_required(f):
       @wraps(f)
       def decorated(*args, **kwargs):
           token = request.cookies.get('token')
           if not token:
               return jsonify({'error': 'Authorization token is missing'}), 401
           try:
               data = jwt.decode(token, app.config['SECRET_KEY'], algorithms=["HS256"])
               current_user_id = data['user_id']
           except DecodeError:
               return jsonify({'error': 'Authorization token is invalid'}), 401
           return f(current_user_id, *args, **kwargs)
       return decorated
   ```

5. Define an API endpoint for user authentication:

   ```python
   with open('users.json', 'r') as f:
       users = json.load(f)

   @app.route('/auth', methods=['POST'])
   def authenticate_user():
       if request.headers['Content-Type'] != 'application/json':
           return jsonify({'error': 'Unsupported Media Type'}), 415
       username = request.json.get('username')
       password = request.json.get('password')
       for user in users:
           if user['username'] == username and user['password'] == password:
               token = jwt.encode({'user_id': user['id']}, app.config['SECRET_KEY'],algorithm="HS256")
               response = make_response(jsonify({'message': 'Authentication successful'}))
               response.set_cookie('token', token)
               return response, 200
       return jsonify({'error': 'Invalid username or password'}), 401
   ```

6. Update the `/products` endpoint to check for JWT:
   ```python
   @app.route('/products', methods=['GET'])
   @token_required
   def get_products(current_user_id):
       headers = {'Authorization': f'Bearer {request.cookies.get("token")}'}
       response = requests.get(f"{BASE_URL}/products", headers=headers)
       if response.status_code != 200:
           return jsonify({'error': response.json()['message']}), response.status_code
       products = []
       for product in response.json()['products']:
           product_data = {
               'id': product['id'],
               'title': product['title'],
               #'brand': product['brand'],
               'price': product['price'],
               'description': product['description']
           }
           products.append(product_data)
       return jsonify({'data': products}), 200 if products else 204
   ```

## Containerizing with Docker

1. Create a `Dockerfile` in the root directory:

   ```dockerfile
   FROM python:3.9-alpine
   WORKDIR /app
   COPY requirements.txt ./
   RUN pip install -r requirements.txt
   COPY . .
   EXPOSE 5000
   CMD ["python", "./services/products.py"]
   ```

2. Add a `.dockerignore` file to the root directory:

   ```
   /venv
   /services/__pycache__/
   .gitignore
   ```

3. Build the Docker image:

   ```bash
   docker build -t flask-microservice .
   ```

4. Run the microservice in a Docker container:
   ```bash
   docker run -p 5000:5000 flask-microservice
   ```

Your microservice should now be running and accessible at `http://localhost:5000`.

References:

https://docs.python.org/3/

https://flask.palletsprojects.com/en/3.0.x/

https://docs.docker.com/reference/

https://kinsta.com/pt/blog/microsservicos-python/
