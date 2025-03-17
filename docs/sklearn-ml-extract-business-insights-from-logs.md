# Using scikit-learn to Extract Business Insights from Website Logs: A Practical Guide

## Introduction

I often hear many organisations hesitate to dive into machine learning (ML) projects due to perceived limitations, such as insufficient data or unclear use cases. However, valuable insights are often hidden in plain sight within existing resources, such as website logs. These logs provide detailed information on user interactions and can offer actionable insights when analysed using ML techniques. This guide will walk you through implementing and deploying a machine learning model with `sklearn` in Python, using website logs to generate business value.

## Prerequisites

Before getting started, ensure you have the following:

1. **Basic Understanding of Python:** Familiarity with Python programming is essential.
2. **Installed Python Environment:** Have Python 3.x installed on your system along with pip.
3. **Basic Knowledge of Machine Learning Concepts:** Understanding of supervised learning, feature extraction, and model evaluation is beneficial.
4. **Access to Website Logs:** You should have access to website log data, similar to the format shared in the example provided. ****I use [Pino](https://www.npmjs.com/package/pino) that formats the output in json****.
5. **Installed Libraries:** Ensure `sklearn`, `pandas`, and `numpy` are installed. You can install these packages using pip:
```bash
pip install sklearn pandas numpy
```

## Use Case

| Aspect | Description |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Problem | Extracting business value from existing website logs using machine learning. |
| Solution | Implementation of a decision tree classifier using scikit-learn to predict user conversions. |
| Data Source | Website logs in JSON format (using Pino logger). |
| Key Features | - User agent information<br>- Status codes<br>- Response times<br>- URL patterns |
| Business Value | - Predict user conversion likelihood<br>- Optimise user experience<br>- Inform marketing strategies<br>- Improve website efficiency |

## Step-by-Step Guide

### Step 1: Data Collection

Start by collecting the website logs. These logs are typically in JSON format and include fields like `url`, `user-agent`, `statusCode`, and response time. Extract these logs from your server and store them in a CSV or JSON for easier manipulation.

### Step 2: Data Preprocessing

Load the logs into a pandas DataFrame:
```python
import pandas as pd

logs_df = pd.read_json('website_logs.json')
```
Convert strings and nested dictionaries into usable features. For example, you can extract browser information from the `user-agent` string.

### Step 3: Feature Engineering

Identify and extract relevant features from your data:
```python
logs_df['browser'] = logs_df['req']['headers'].apply(lambda h: h.get('user-agent', ''))
logs_df['status'] = logs_df['res']['statusCode']
logs_df['response_time'] = logs_df['responseTime']
```

### Step 4: Model Selection

Choose a suitable ML model from `sklearn`. For simplicity, let's use a decision tree to predict if a session leads to a certain user action, like a conversion:
```python
from sklearn.model_selection import train_test_split
from sklearn.tree import DecisionTreeClassifier

# Define features and target
features = logs_df[['status', 'response_time']]
target = (logs_df['url'] == '/conversion-url').astype(int)

# Split data
X_train, X_test, y_train, y_test = train_test_split(features, target, test_size=0.2, random_state=42)

# Create model
clf = DecisionTreeClassifier()
clf.fit(X_train, y_train)
```

### Step 5: Model Evaluation

Evaluate the performance of your model:
```python
from sklearn.metrics import accuracy_score

predictions = clf.predict(X_test)
accuracy = accuracy_score(y_test, predictions)
print(f'Model accuracy: {accuracy}')
```

### Step 6: Model Deployment

Once satisfied with the model performance, deploy it using a Flask API for real-time inference:
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/predict', methods=['POST'])
def predict():
  data = request.get_json(force=True)
  features = [data['status'], data['response_time']]
  prediction = clf.predict([features])
  return jsonify({'conversion_prediction': bool(prediction)})
```
Run this server and connect it to your website to process log data in real-time.

## How to Use the Model and Deploy It in Production

Deploy the model in your production environment and use it to process incoming web logs. The prediction can help optimise user experiences by identifying behaviours linked to successful conversions. Use this information to shape marketing strategies, tailor user experiences, and improve overall website efficiency.

## Conclusion

By leveraging website logs, you unlock a wealth of data that can fuel machine learning projects and provide tangible business value. Using `sklearn`, you can quickly implement models to extract insights, allowing your organisation to make data-driven decisions and optimise its operations. By following this guide, you will have laid the foundation for future expansion into more complex ML systems.

Now go forth and explore what treasures are hidden in your Databases, Applications, Security Events etc. logs.
