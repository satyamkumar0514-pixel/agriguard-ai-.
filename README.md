1. Project Structure

codeText

annrakshak-ai/
├── backend/
│   ├── main.py            # API Endpoints
│   ├── model_loader.py    # AI Inference Logic
│   ├── database.py        # SQLAlchemy Models
│   ├── schemas.py         # Pydantic Data Validation
│   └── models/            # Saved .h5 or .keras models
├── frontend/
│   ├── src/
│   │   ├── components/    # UI Components (Upload, Dashboard, History)
│   │   ├── pages/         # Landing, Detection, Result
│   │   └── App.js
├── requirements.txt
└── README.md

2. Backend Implementation (FastAPI)

backend/main.py

This handles the image upload, triggers AI analysis, and saves the result to the history.

codePython

from fastapi import FastAPI, UploadFile, File, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
import shutil
import os
from datetime import datetime
from model_loader import predict_image
from database import SessionLocal, PredictionRecord

app = FastAPI(title="AgriGuard AI API")

# Enable CORS for React
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/predict")
async def predict(crop_type: str, file: UploadFile = File(...), db=Depends(get_db)):
    # 1. Save uploaded file temporarily
    file_path = f"temp_{file.filename}"
    with open(file_path, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    try:
        # 2. Run AI Inference
        result = predict_image(file_path, crop_type)
        
        # 3. Save to Database History
        new_record = PredictionRecord(
            crop=crop_type,
            disease=result['disease'],
            confidence=result['confidence'],
            status="Diseased" if result['disease'] != "Healthy" else "Healthy",
            created_at=datetime.now()
        )
        db.add(new_record)
        db.commit()
        
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
    finally:
        if os.path.exists(file_path):
            os.remove(file_path)

@app.get("/history")
async def get_history(db=Depends(get_db)):
    return db.query(PredictionRecord).order_by(PredictionRecord.created_at.desc()).all()

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)

backend/model_loader.py

Pre-processing and AI Prediction using TensorFlow.

codePython

import tensorflow as tf
import numpy as np
from PIL import Image

# Metadata mapping (In production, load this from a JSON/DB)
DISEASE_INFO = {
    "Tomato_Early_Blight": {
        "symptoms": "Dark spots with concentric rings on older leaves.",
        "prevention": "Rotate crops and maintain leaf dryness.",
        "treatment": "Apply copper-based fungicides."
    },
    "Healthy": {
        "symptoms": "None", "prevention": "Continue standard care.", "treatment": "N/A"
    }
}

# Load Pre-trained Model (PlantVillage Dataset recommended)
# model = tf.keras.models.load_model('models/plant_disease_model.h5')

def predict_image(img_path, crop):
    # Image Preprocessing
    img = Image.open(img_path).resize((224, 224))
    img_array = np.array(img) / 255.0
    img_array = np.expand_dims(img_array, axis=0)

    # Simulated Prediction (Replace with model.predict)
    # predictions = model.predict(img_array)
    # class_idx = np.argmax(predictions)
    
    # Mock Response for Prototype
    confidence = 0.94
    predicted_disease = "Tomato_Early_Blight" if crop == "Tomato" else "Healthy"
    
    info = DISEASE_INFO.get(predicted_disease, DISEASE_INFO["Healthy"])
    
    return {
        "crop": crop,
        "disease": predicted_disease.replace("_", " "),
        "confidence": float(confidence),
        "status": "Diseased" if "Healthy" not in predicted_disease else "Healthy",
        "details": info
    }

3. Frontend Implementation (React + Tailwind)

frontend/src/App.js (The Main UI)

codeJsx

import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AgriGuard = () => {
  const [selectedFile, setSelectedFile] = useState(null);
  const [preview, setPreview] = useState(null);
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);
  const [crop, setCrop] = useState("Tomato");

  const handleFileChange = (e) => {
    const file = e.target.files[0];
    setSelectedFile(file);
    setPreview(URL.createObjectURL(file));
  };

  const analyzeCrop = async () => {
    if (!selectedFile) return;
    setLoading(true);
    const formData = new FormData();
    formData.append('file', selectedFile);
    
    try {
      const response = await axios.post(`http://localhost:8000/predict?crop_type=${crop}`, formData);
      setResult(response.data);
    } catch (error) {
      alert("Error analyzing image");
    }
    setLoading(false);
  };

  return (
    <div className="min-h-screen bg-stone-50 font-sans text-slate-900">
      {/* Navigation */}
      <nav className="bg-emerald-700 text-white p-4 shadow-lg">
        <div className="max-w-6xl mx-auto flex justify-between items-center">
          <h1 className="text-2xl font-bold flex items-center gap-2">
            🌿 AgriGuard AI
          </h1>
          <div className="space-x-6">
            <button className="hover:text-emerald-200">Dashboard</button>
            <button className="hover:text-emerald-200">History</button>
            <button className="bg-white text-emerald-700 px-4 py-2 rounded-lg font-semibold">Login</button>
          </div>
        </div>
      </nav>

      {/* Hero Section */}
      <main className="max-w-4xl mx-auto py-12 px-4">
        <section className="text-center mb-12">
          <h2 className="text-4xl font-extrabold text-slate-800 mb-4">Identify Crop Diseases Instantly</h2>
          <p className="text-lg text-slate-600">Upload a photo of your crop leaf to get AI-powered diagnosis and treatment advice.</p>
        </section>

        {/* Prediction Card */}
        <div className="bg-white rounded-3xl shadow-xl p-8 border border-stone-200">
          <div className="grid md:grid-cols-2 gap-8">
            {/* Upload Area */}
            <div className="space-y-4">
              <label className="block text-sm font-medium text-gray-700">Select Crop Type</label>
              <select 
                className="w-full p-3 border rounded-xl bg-stone-50"
                value={crop}
                onChange={(e) => setCrop(e.target.value)}
              >
                <option>Tomato</option>
                <option>Potato</option>
                <option>Rice</option>
                <option>Maize</option>
              </select>

              <div className="border-2 border-dashed border-emerald-200 rounded-2xl h-64 flex flex-col items-center justify-center bg-emerald-50/30 overflow-hidden relative">
                {preview ? (
                  <img src={preview} className="absolute inset-0 w-full h-full object-cover" alt="Preview" />
                ) : (
                  <div className="text-center p-4">
                    <span className="text-4xl">📸</span>
                    <p className="mt-2 text-sm text-emerald-800 font-medium">Click to upload or drag leaf image</p>
                  </div>
                )}
                <input type="file" className="absolute inset-0 opacity-0 cursor-pointer" onChange={handleFileChange} />
              </div>
              
              <button 
                onClick={analyzeCrop}
                disabled={loading || !selectedFile}
                className="w-full bg-emerald-600 hover:bg-emerald-700 text-white font-bold py-4 rounded-xl transition-all shadow-md disabled:bg-slate-300"
              >
                {loading ? "Analyzing Matrix..." : "Analyze Crop Health"}
              </button>
            </div>

            {/* Results Area */}
            <div className="bg-stone-50 rounded-2xl p-6 border border-stone-100">
              {result ? (
                <div className="space-y-4 animate-in fade-in duration-500">
                  <div className={`inline-block px-4 py-1 rounded-full text-sm font-bold ${result.status === 'Healthy' ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
                    {result.status === 'Healthy' ? '🟢 Healthy Plant' : '🔴 Disease Detected'}
                  </div>
                  <h3 className="text-2xl font-bold">{result.disease}</h3>
                  <div className="flex items-center gap-2">
                    <span className="text-gray-500">Confidence:</span>
                    <div className="w-full bg-gray-200 rounded-full h-2">
                      <div className="bg-emerald-500 h-2 rounded-full" style={{width: `${result.confidence * 100}%`}}></div>
                    </div>
                    <span className="font-bold text-emerald-600">{(result.confidence * 100).toFixed(0)}%</span>
                  </div>
                  
                  <div className="mt-6 space-y-4">
                    <div className="bg-white p-4 rounded-xl shadow-sm">
                      <h4 className="font-bold text-emerald-800">Symptoms</h4>
                      <p className="text-sm text-gray-600">{result.details.symptoms}</p>
                    </div>
                    <div className="bg-white p-4 rounded-xl shadow-sm">
                      <h4 className="font-bold text-emerald-800">Treatment</h4>
                      <p className="text-sm text-gray-600">{result.details.treatment}</p>
                    </div>
                  </div>
                </div>
              ) : (
                <div className="h-full flex items-center justify-center text-slate-400 italic">
                  Analysis results will appear here...
                </div>
              )}
            </div>
          </div>
        </div>
      </main>
    </div>
  );
};

export default AgriGuard;

4. Database Schema (SQLAlchemy)

Save this in backend/database.py.

codePython

from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./agriguard.db"

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class PredictionRecord(Base):
    __tablename__ = "predictions"
    id = Column(Integer, primary_key=True, index=True)
    crop = Column(String)
    disease = Column(String)
    confidence = Column(Float)
    status = Column(String)
    created_at = Column(DateTime)

Base.metadata.create_all(bind=engine)

5. Setup & Training Instructions

Training the Model

Dataset: Use the PlantVillage Dataset (available on Kaggle). It contains 54,000+ images of 14 crop species.

Architecture: Use MobileNetV2 for transfer learning because it is lightweight and runs fast on mobile/web servers.

Command to install dependencies:

codeBash

pip install fastapi uvicorn tensorflow pillow sqlalchemy python-multipart

Running the System

Start Backend:

codeBash

cd backend
python main.py

Start Frontend:

codeBash

cd frontend
npm install axios
npm start

6. Key AI Strategies for Reliability

Confidence Threshold: In model_loader.py, if confidence < 0.70, the system should return a "Low Confidence" warning asking for a clearer image.

Data Augmentation: During model training, use random rotation, zooms, and horizontal flips to ensure the AI works even if the farmer captures the photo at an angle.

Normalization: Ensure the image input is normalized to [0, 1] or [-1, 1] to match the pre-trained MobileNet expectations.

This prototype provides a functional foundation. For production, you would replace the "Mock Response" with the actual .h5 model loading and add a multi-language translation layer (e.g., react-i18next) for Hindi support
