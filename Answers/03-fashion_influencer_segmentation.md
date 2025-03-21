

# 3rd Question - Advanced Fashion Influencer Segmentation
## The Question
We would like to push our segmentation one step further. The current methodology refrains us
from spotting interesting (sub)categories of accounts like fashion enthusiasts, luxury,
sportswear, etc. Several families of methods can be considered to improve this methodology:

- Statistical approach
- Machine Learning approaches (supervised and/or unsupervised)
- Hybrid approach combining deterministic rules and machine learning

Which approach would you prefer and why? What would be the advantages/limits of each of
these methods? You are free to exploit the author database the way you want. You can also
generate new tables to develop your approach.

[Optional] In case you have ideas of approaches that would require more time that you are
given to perform this test, it will be appreciated if you mention them and explain their added
value / limitations.

## What We Understood

1. **Current Segmentation Limitations:**
    - The current methodology is based solely on follower counts, which does not capture nuanced subcategories like fashion enthusiasts or luxury influencers.
    - Available data includes:
        - `MART_AUTHORS`: Author metadata.
        - `MART_AUTHORS_SEGMENTATIONS`: Current segmentation data.
        - `MART_IMAGES_LABELS`: Image analysis results.
        - `MART_IMAGES_OF_POSTS`: Links posts to images.
2. **Task:**
    - Evaluate and improve the segmentation methodology to identify subcategories like luxury or sportswear influencers.

---

## Result and Step-by-Step Explanation

### 1. Machine Learning Approach (Unsupervised)

#### **Step 1: Data Preparation**

- Extract features from images and captions using deep learning models (e.g., EfficientNet for images, BERT for captions).
- Create a multi-modal feature matrix combining visual, textual, and engagement data.

```python
from transformers import BertTokenizer, BertModel
from torchvision import models

# Load pre-trained models
tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
bert_model = BertModel.from_pretrained('bert-base-uncased')
vision_model = models.EfficientNet_V2_S(pretrained=True)

# Function to extract features
def extract_features(caption, image):
    # Text features
    inputs = tokenizer(caption, return_tensors="pt")
    outputs = bert_model(**inputs)
    text_features = outputs.last_hidden_state[:, 0, :].detach().numpy()[0]
    
    # Image features
    image_features = vision_model(image).detach().numpy()[0]
    
    return np.concatenate((text_features, image_features))

# Apply to dataset
features = []
for row in dataset:
    caption = row['caption']
    image = row['image']
    features.append(extract_features(caption, image))
```


#### **Step 2: Clustering**

- Use unsupervised clustering algorithms (e.g., HDBSCAN) to identify clusters representing different subcategories.
- Analyze cluster characteristics to label them (e.g., "Luxury Fashion" or "Sportswear Enthusiasts").

```python
import hdbscan

# Perform clustering
clusterer = hdbscan.HDBSCAN(min_samples=50, min_cluster_size=100)
cluster_labels = clusterer.fit_predict(features)

# Analyze clusters
for cluster_id in np.unique(cluster_labels):
    cluster_features = features[cluster_labels == cluster_id]
    # Calculate average engagement, top labels, etc.
    avg_engagement = np.mean([f['engagement'] for f in cluster_features])
    top_labels = Counter([f['label'] for f in cluster_features]).most_common(5)
    print(f"Cluster {cluster_id}: {top_labels}, Avg Engagement: {avg_engagement}")
```


#### **Advantages:**

- **Pattern Discovery:** Finds hidden subcategories without manual labeling.
- **Flexibility:** Adapts to new trends and data changes.
- **Data-Driven:** Removes human bias from segmentation.


#### **Limitations:**

- **Interpretability:** Harder to explain to non-technical stakeholders.
- **Expertise Required:** Needs machine learning expertise.
- **Continuous Training:** Requires regular retraining as data evolves.

---

### 2. Hybrid Approach Combining Deterministic Rules and Machine Learning

#### **Step 1: Rule-Based Pre-Filtering**

- Define deterministic rules to filter out irrelevant accounts (e.g., those with low engagement or no fashion interest).
- Use SQL to apply these rules and create a filtered dataset.

```sql
CREATE TABLE FILTERED_AUTHORS AS
SELECT *
FROM MART_AUTHORS
WHERE NB_FOLLOWERS > 1000 AND FASHION_INTEREST_SEGMENT = TRUE;
```


#### **Step 2: Machine Learning Clustering**

- Apply unsupervised clustering (e.g., HDBSCAN) on the filtered dataset to discover nuanced subcategories.
- Use the same clustering analysis as in the ML approach to label clusters.

```python
# Perform clustering on filtered dataset
filtered_features = [extract_features(row['caption'], row['image']) for row in filtered_authors]
cluster_labels_filtered = clusterer.fit_predict(filtered_features)

# Analyze clusters
for cluster_id in np.unique(cluster_labels_filtered):
    cluster_features_filtered = filtered_features[cluster_labels_filtered == cluster_id]
    # Calculate average engagement, top labels, etc.
    avg_engagement = np.mean([f['engagement'] for f in cluster_features_filtered])
    top_labels = Counter([f['label'] for f in cluster_features_filtered]).most_common(5)
    print(f"Cluster {cluster_id}: {top_labels}, Avg Engagement: {avg_engagement}")
```


#### **Step 3: Neural Network**

- Implement a neuro-symbolic layer that integrates deterministic rules with learned representations.
- Use differentiable logic to make rules trainable and adaptive.

1- Deterministic Rules Layer:
  - Define clear rules based on domain knowledge (e.g., "Luxury" = presence of "Hermes" or "Chanel").
  - These rules act as a filter to ensure only relevant accounts are considered.

2- Neural Network Layer:
  - Train a neural network to learn complex patterns from the data that the rules might miss.

  - This layer should be able to identify nuanced relationships between different fashion items or trends.

3- Integration:
  - integrate the two layers.




#### **Advantages:**

- **Precision:** Rules ensure basic quality while ML finds deeper patterns.
- **Flexibility:** Adapts to new trends while maintaining explainability.
- **Practicality:** Easier to maintain than pure ML while still being adaptive.


#### **Limitations:**

- **Complexity:** More complex to implement than purely statistical methods.
- **Expertise Required:** Needs both domain expertise (for rules) and technical expertise (for ML).

---

## Response

Given the need to identify nuanced subcategories like fashion enthusiasts or luxury influencers, I would prefer a **Hybrid Approach** combining deterministic rules with machine learning. This approach offers the best balance between precision, adaptability, and interpretability.

**Why Hybrid?**

- **Precision:** Deterministic rules ensure that only relevant accounts are considered, reducing noise and improving segmentation quality.
- **Adaptability:** Machine learning components adapt to new trends and patterns in the data, allowing for real-time discovery of emerging subcategories.
- **Interpretability:** The hybrid approach maintains transparency by integrating explainable rules with learned patterns, making it easier for stakeholders to understand segmentation decisions.

While machine learning approaches are powerful for discovering hidden patterns, they lack the interpretability and precision that deterministic rules provide. The hybrid approach bridges this gap, offering a robust and flexible methodology for advanced segmentation tasks.

**Optional Advanced Methods:**

- **Deep Learning Vision Analysis:** Use CNNs to analyze images for style, color palette, or outfit complexity.
- **Temporal Behavior Tracking:** Implement time-series analysis to identify trendsetters vs followers and track segment evolution over time.

These methods would further enhance segmentation by providing deeper insights into visual trends and temporal behavior, but they require additional resources and time to implement effectively.

