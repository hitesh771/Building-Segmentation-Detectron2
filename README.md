# Building-Segmentation-Detectron2
🏙️ Automated Building Extraction with Mask R-CNN
This repository contains the complete end-to-end pipeline for training a Mask R-CNN model to perform instance segmentation on aerial satellite imagery.

The primary goal is to take any top-down satellite image, identify every building, and extract each one into a separate, transparent .png file.

🚀 The Core Problem This Solves
Many high-quality satellite datasets (like the WHU dataset) are in the U-Net format (Image + Mask files). However, advanced models like Mask R-CNN require the COCO format (a single _annotations.json file).

This project's centerpiece is a custom conversion script that bridges this gap, enabling you to train a state-of-the-art Detectron2 model on a U-Net dataset.

✨ Features
Data Conversion: A Python script that converts a U-Net style (Image/Mask) dataset into the COCO .json format required by Detectron2.

Transfer Learning: The model is fine-tuned using transfer learning from the COCO pre-trained model for high accuracy and fast training.

Inference App: A final notebook that allows you to upload any satellite image and receive a .zip file of all extracted buildings.

💻 Tech Stack
Python 3.10+

PyTorch

Detectron2 (for Mask R-CNN)

OpenCV (for image processing & contour finding)

Google Colab (for T4 GPU-accelerated training)

NumPy & Matplotlib

📈 Results: The Training Curve
The model was trained for 1,000 iterations on a T4 GPU. The loss curve demonstrates successful convergence as the model learned to identify building footprints.

🚀 How to Use the Final App (Inference)
This notebook allows you to use the pre-trained model to extract buildings from any image.

Open 2_Inference_App.ipynb in Google Colab (and ensure the runtime is set to T4 GPU).

Add Your Model: Make sure your trained my_building_model.pth file is in your Google Drive root. (You can download my pre-trained model [here](https://tinyurl.com/my-dataset).

Run the Cells: The notebook will install Detectron2, connect to your Drive, and load the model.

Upload: When prompted, upload any satellite image.

Get Results: The app will display the segmented image and automatically download a building_results.zip file containing all the extracted building .png files.

🛠️ How to Re-Train the Model from Scratch
Follow these steps to reproduce the entire training pipeline.

1. Get the Data
Download the WHU Building Dataset from [Kaggle](https://sl1nk.com/kaggledataset).

Unzip the file. You will have a WHU folder.

2. Set Up Your Google Drive
Create a folder in your Google Drive named WHOdataset.

Upload the WHU folder into it.

The final path structure must be: My Drive/WHOdataset/WHU/train/Image and My Drive/WHOdataset/WHU/train/Mask.

3. Run the Conversion
Open the 1_Data_Conversion.ipynb notebook in Google Colab.

Run the cells. This will mount your Google Drive and run the conversion script.

This process will take a long time (1+ hour) as it traces the contours for all 5,700+ images.

Output: It will create the _annotations.json file inside your /WHOdataset/WHU/train/ folder.

4. Run the Training
Open the 2_Training_Notebook.ipynb in Google Colab (with a T4 GPU).

Run all the cells.

Registration: The script will first register your new dataset with Detectron2.

Training: It will then configure and run the training (approx. 7-10 minutes for 1,000 iterations).

Save Model: The final model_final.pth will be saved to the Colab runtime and then automatically copied to your Google Drive as my_building_model.pth.

License
This project is open-sourced under the MIT License. See the LICENSE file for more details.
