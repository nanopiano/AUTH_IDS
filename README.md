# AUTH_IDS
This is the experimental setup used to evaluate the Tier 2 component of the proposed system . Tier 2 sits at the gateway-to-backend layer and is responsible for detecting attacks using machine learning. In this work only the Tier 2 detection layer is implemented and tested, covering the ensemble classifier, the LSTM model, and the G-KDE anomaly detector.

# Network Intrusion Detection with Ensemble, G-KDE, and LSTM Fusion

This project implements a multi-layered intrusion detection system designed to identify various authentication-relevant network attacks using a combination of ensemble learning, Gaussian Kernel Density Estimation (G-KDE) for anomaly detection, and Long Short-Term Memory (LSTM) networks for temporal analysis. The system fuses predictions from these diverse models to achieve robust detection capabilities, prioritizing the reduction of false negatives.

## Project Structure

- `dataset/`: Contains the network traffic datasets used for training and testing.
- `notebook.ipynb`: The main Jupyter notebook detailing the data loading, preprocessing, model training, evaluation, and interpretability.

## Methodology

### 1. Data Loading and Preprocessing

Network traffic data, encompassing both benign and various attack types (ARP Spoofing, MQTT-DDoS, Reconnaissance, TCP/IP SYN attacks), is loaded from `.pcap.csv` files. Key preprocessing steps include:

- **Merging**: Combining individual attack and benign datasets into comprehensive training and testing DataFrames.
- **Labeling**: Assigning standardized labels (e.g., 'spoofing', 'mqtt_auth_attack', 'recon', 'syn_attack') based on filenames and mapping them to a single 'benign' or 'attack' binary classification for specific models.
- **Filtering**: Focusing on a subset of 'authentication-relevant' attack types to streamline the threat model.
- **Scaling**: Standardizing numerical features using `StandardScaler`.
- **Encoding**: Converting categorical labels into numerical format using `LabelEncoder`.

### 2. Model Components

The system employs three distinct models, each contributing to the overall detection capability:

#### a. Ensemble Classifier

A traditional machine learning ensemble built from:
- **Random Forest Classifier**
- **AdaBoost Classifier**
- **Bagging Classifier (with Decision Trees as base estimators)**

These models are trained on the full feature set for multi-class classification, and their predictions are combined using a majority voting scheme.

#### b. G-KDE (Gaussian Kernel Density Estimation) for Anomaly Detection

- **Purpose**: Designed to detect anomalous behavior by learning the density of 'benign' traffic.
- **Feature Selection**: Utilizes the top features identified by the Random Forest model's importance scores to focus the KDE analysis.
- **Training**: Fitted exclusively on benign training data, with a dedicated validation split to determine an optimal anomaly threshold (e.g., 5th percentile of benign scores).
- **Output**: Provides a binary prediction (0: benign, 1: anomaly) based on whether a sample's likelihood score falls below the learned threshold.

#### c. LSTM (Long Short-Term Memory) for Temporal Analysis

- **Purpose**: Specialized for detecting attacks with temporal patterns, specifically targeting MQTT-related authentication floods.
- **Subset Training**: Trained only on the MQTT-related benign and attack traffic subset.
- **Sequence Construction**: Data is transformed into sequences using a sliding window approach to capture temporal dependencies.
- **Architecture**: A `Sequential` Keras model with multiple LSTM layers and Dropout for regularization, followed by a Dense output layer with sigmoid activation for binary classification.

### 3. Fusion Strategy

To maximize attack detection rates (minimize False Negatives), the binary predictions from the Ensemble, G-KDE, and LSTM models are combined using **OR-logic**. If any of the three models classify a sample as an 'attack', the final fused prediction is 'attack'.

### 4. Evaluation

The performance of individual models and the fused system is rigorously evaluated using a custom `evaluate_binary_model` function, which provides consistent metrics for binary classification (attack vs. benign):

- **Metrics**: Accuracy, Precision, Recall, F1-Score, False Positive Rate (FPR), False Negative Rate (FNR), and Area Under the ROC Curve (AUC).
- **Visualizations**: Confusion matrices provide a clear breakdown of true positives, true negatives, false positives, and false negatives. ROC curves illustrate the trade-off between True Positive Rate and False Positive Rate for comparable models.

### 5. Interpretability (SHAP)

SHAP (SHapley Additive exPlanations) values are used to interpret the feature importance and impact within the Random Forest component of the ensemble. This helps in understanding which network traffic features are most influential in driving the model's predictions for different attack types.
