---
title: "ML Project | Network Intrusion Detection System"
date: 2024-12-11 14:00:00 +0100
categories: [Machine Learning, Projects]
tags: [ml, ids, random-forest, cicids2017, deployment, streamlit, huggingface]
image:
  path: https://github.com/user-attachments/assets/d22f99e8-d1be-4ed7-a0e5-8fa32545a9aa
---

# Summary

This was my project for the Machine Learning course at IT Business School. I trained a binary classifier on the CICIDS2017 dataset that takes a network flow and labels it as benign or attack. I tried three models: a Random Forest, a neural network and logistic regression. The Random Forest did best with 99.96% accuracy on the test set, so that is the one I deployed.

The code and the trained model are on [GitHub](https://github.com/ampls-911/ML_streamlit). You can try the model in the [Streamlit app](https://mlapp-vjxvm6hnwohgftvqh4ep8g.streamlit.app/) or the [Hugging Face Space](https://huggingface.co/spaces/ampls/ML-Project-IDS).

# Dataset

CICIDS2017 has about 2.8 million labelled network flows with 78 features each. The original labels name the attack type, but I merged them all into a single ATTACK class, so the problem is BENIGN vs ATTACK.

The raw data needed cleaning first. It contains missing values, infinite values and outliers, and I dealt with those before doing anything else.

The dataset is also very imbalanced, with far more benign flows than attacks. I undersampled the benign class and ended up with 791,484 flows, 50% benign and 50% attack. I then split that 70/15/15 into training, validation and test sets.

# Models

I trained three models on the same split:

- Random Forest with 200 trees (scikit-learn)
- Neural network with three hidden layers of 256, 128 and 64 units (TensorFlow/Keras), with early stopping
- Logistic regression as a baseline

I used 5-fold cross-validation during training. The test set was only used for the final evaluation.

# Results

| Model | Accuracy | Precision | Recall | F1-Score |
|-------|----------|-----------|--------|----------|
| **Random Forest** | **99.96%** | **0.9996** | **0.9996** | **0.9996** |
| Neural Network | 99.46% | 0.9946 | 0.9943 | 0.9945 |
| Logistic Regression | 94.05% | 0.9351 | 0.9517 | 0.9433 |

Logistic regression already gets 94%, so the two classes are fairly easy to separate with these features. The neural network and the Random Forest are both above 99%, with the Random Forest slightly ahead. I ran McNemar's test to check that the difference between the models is significant, and it is (p < 0.001).

The test set has 118,728 flows. The Random Forest got 51 of them wrong: 12 attacks were classified as benign, and 39 benign flows were flagged as attacks.

The 12 missed attacks are the worse kind of error for an IDS, since those are attacks that go through unnoticed. The 39 false alarms matter too. That is about 330 per million flows, and on a busy network that many false alerts would be a lot for an analyst to go through.

# Limitations

99.96% looks great, but I would not expect this model to do that well on a real network.

The main reason is the balancing. My test set is half attacks, while real traffic is almost all benign. On real traffic most of the alerts would be false alarms, even with the same error rates.

The training and test data also come from the same capture, meaning the same network and the same week in 2017. I have not tested the model on traffic from anywhere else, so I don't know how well it generalizes.

CICIDS2017 itself has known issues. Engelen et al. found labelling and flow construction errors in it ("Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study", 2021), and very high scores on this dataset are common in other papers too.

Finally, the model only says attack or benign, not which attack, and I did not test it against traffic that is crafted to evade detection.

# Deployment

I deployed the Random Forest on two platforms.

The Streamlit app lets you upload a CSV of flows and get predictions for all of them, which you can then download. It also shows the model's metrics and the feature importances.

The Hugging Face Space uses Gradio and has three tabs: model info, batch prediction and a usage guide. The model file is too big for a normal git repo, so it is stored with Git LFS.

Both apps expect a CSV of flow features, not raw packets, so they are demos and not something you can connect to a live network.

# Future work

- Test on imbalanced data that is closer to real traffic
- Multi-class classification, to identify the attack type
- Adversarial robustness testing
- Real-time processing and integration with a network monitoring tool

*Project supervised by Prof. Ahmed Ben Taleb, Machine Learning course, IT Business School, Semester 1, 2024-2025.*
