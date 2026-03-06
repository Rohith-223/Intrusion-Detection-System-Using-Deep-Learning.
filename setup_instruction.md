# Intrusion Detection System Using Deep Learning
## Setup Instructions

This document explains how to install and run the Intrusion Detection System (IDS) project.

---

## 1. Prerequisites

Make sure the following software is installed:

- Python 3.9 or higher
- pip (Python package manager)
- Git
- Web browser

Recommended IDE:
- Visual Studio Code

---

## 2. Clone the Repository

Open terminal or command prompt and run:

git clone https://github.com/Rohith-223/Intrusion-Detection-System-Using-Deep-Learning.git

Then move into the project directory:

cd Intrusion-Detection-System-Using-Deep-Learning

---

## 3. Create Virtual Environment

Create a virtual environment to manage dependencies.

Windows:

python -m venv venv

Activate environment:

venv\Scripts\activate

Linux / Mac:

python3 -m venv venv
source venv/bin/activate

---

## 4. Install Dependencies

Install required Python libraries from requirement.txt.

pip install -r requirement.txt

Libraries include:

- TensorFlow
- Keras
- Pandas
- NumPy
- Scikit-learn
- FastAPI
- Uvicorn

---

## 5. Run the Backend Server

Start the FastAPI backend server:

uvicorn main:app --reload

The server will start at:

http://127.0.0.1:8000

---

## 6. Open API Documentation

FastAPI automatically generates API documentation.

Open browser and go to:

http://127.0.0.1:8000/docs

---

## 7. Run the Frontend Dashboard

Navigate to the frontend folder:

cd frontend

Start the frontend server:

python -m http.server 3000

Open the dashboard in browser:

http://localhost:3000

---

## 8. Upload Dataset

Use the dashboard to upload the CICIDS dataset CSV file.

The system will classify network traffic into:

- Benign
- DoS
- DDoS
- PortScan
- Brute Force
- Botnet

---

## 9. Output

The system will display:

- Intrusion detection result
- Traffic classification
- Attack distribution charts

---

## 10. Stop the Server

Press:

CTRL + C

to stop the backend or frontend server.

---

## Author

Rohith  
GitHub: https://github.com/Rohith-223
