
## 📋 The Question

Propose a new segmentation methodology for fashion trends analysis, addressing:
1. What kind of analysis would you do to evaluate the quality of a segmentation methodology?
2. What biases can you identify in the panel creation or in the accounts selection that might distort insights and how would you correct them?

## 🧠 What We Understood

1. **Current Segmentation:**
   - Based solely on follower counts:
     - Mainstream: ≤12,000 followers
     - Trendy: 12,001-40,000 followers
     - Edgy: >40,000 followers
   - Stored in `MART_AUTHORS_SEGMENTATIONS`

2. **Available Data:**
   - `MART_AUTHORS`: Author metadata
   - `MART_AUTHORS_SEGMENTATIONS`: Current segmentation data
   - `MART_IMAGES_LABELS`: Image analysis results with types: shoe, object_detection, pattern, shoe_brand, bag_brand
   - `MART_IMAGES_OF_POSTS`: Links posts to images

```sql
-- Authors
MART_AUTHORS(AUTHORID, BIO, NB_FOLLOWERS, NB_FOLLOWS)

-- Segmentation
MART_AUTHORS_SEGMENTATIONS(AUTHORID, GEO_ZONE, NB_FOLLOWERS, FASHION_INTEREST_SEGMENT)

-- Image Labels
MART_IMAGES_LABELS(IMAGE_ID, TYPE, POSITION_IN_IMAGE, LABEL_NAME)

-- Post Relationships
MART_IMAGES_OF_POSTS(POST_ID,
    AUTHORID,
	COMMENT_COUNT,
	POST_CAPTION,
	NB_LIKES,
	POST_PUBLICATION_DATE,
	IMAGE_ID)
```

3. **Core Challenge:** 
   - Follower-only segmentation misses crucial fashion-specific insights
   - Need for a more nuanced, style-based approach
   - The challenge is to move beyond follower-only segmentation to a more nuanced, fashion-focused approach

## 🔍 Results and Step-by-Step Explanation
### What kind of analysis would you do to evaluate the quality of a segmentation methodology?

To evaluate the quality of a segmentation methodology, I would perform the following analyses:

#### Step 1: Intra-Segment Homogeneity Analysis

- Calculate average engagement rates, standard deviations, and coefficients of variation for each segment.
- Lower coefficient of variation indicates better homogeneity within a segment.

Example query and results:

```sql
WITH SegmentStats AS
  (SELECT CASE
              WHEN a.NB_FOLLOWERS <= 12000 THEN 'Mainstream'
              WHEN a.NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
              ELSE 'Edgy'
          END AS SEGMENT,
          AVG(p.NB_LIKES / NULLIF(a.NB_FOLLOWERS, 0)) AS Avg_Engagement_Rate,
          STDDEV(p.NB_LIKES / NULLIF(a.NB_FOLLOWERS, 0)) AS Engagement_Rate_StdDev,
          COUNT(DISTINCT a.AUTHORID) AS Author_Count
   FROM MART_AUTHORS a
   JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
   GROUP BY SEGMENT)
SELECT SEGMENT,
       Avg_Engagement_Rate,
       Engagement_Rate_StdDev,
       Engagement_Rate_StdDev / Avg_Engagement_Rate AS Coefficient_of_Variation,
       Author_Count
FROM SegmentStats
ORDER BY SEGMENT;

```

Explanation:

| **Segment**   | **Avg Engagement Rate** | **Engagement Rate StdDev** | **Coefficient of Variation** | **Author Count** |
|---------------|-------------------------|----------------------------|------------------------------|------------------|
| Edgy          | 0.0133                  | 0.1258                     | 9.4846                       | 1206             |
| Mainstream    | 0.0389                  | 2.0755                     | 53.3689                      | 16986            |
| Trendy        | 0.0209                  | 0.3329                     | 15.9154                      | 1296             |

- **Avg Engagement Rate**: This is the average engagement rate for each segment (i.e., the number of likes divided by the number of followers). Higher numbers indicate more active engagement.
  - **Mainstream** authors have the highest average engagement rate, followed by **Trendy**, then **Edgy** authors.
  
- **Engagement Rate StdDev**: This is the standard deviation of the engagement rate, showing how much variation exists in the engagement rates within each segment.
  - **Mainstream** authors have the largest variation, indicating that there is a wide range of engagement rates among authors with a smaller follower count.
  - **Edgy** authors, with a large follower base, have the smallest variation.

- **Coefficient of Variation**: This is the ratio of the standard deviation to the average engagement rate, giving a sense of relative variability.
  - **Mainstream** authors have the highest coefficient of variation, which means their engagement rates are highly inconsistent compared to their average rate.
  - **Edgy** authors are more consistent in their engagement rates, as shown by their lower coefficient of variation.

- **Author Count**: This shows the number of distinct authors in each segment.
  - The **Mainstream** segment has by far the largest number of authors, followed by **Trendy**, then **Edgy**. This suggests that authors with fewer followers are more common.

Conclusion : 
- The Edgy segment appears to be the most homogeneous, which is good for segmentation.

- The Mainstream segment shows high variability, suggesting it might be too broad and could benefit from further subdivision.

- The Trendy segment falls in between, showing moderate homogeneity.



#### Step 2: Inter-Segment Distinctiveness

What we are trying to achieve :
- Compare segments pairwise on key metrics :
    - Larger differences indicate more distinct segments
    - Small differences suggest segments might not be meaningfully different


Analysis:
```sql
WITH SegmentProfiles AS
  (SELECT CASE
              WHEN NB_FOLLOWERS <= 12000 THEN 'Mainstream'
              WHEN NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
              ELSE 'Edgy'
          END AS SEGMENT,
          AVG(NB_LIKES / NULLIF(NB_FOLLOWERS, 0)) AS Avg_Engagement_Rate,
          COUNT(DISTINCT l.TYPE) AS Unique_Fashion_Items,
          COUNT(DISTINCT l.LABEL_NAME) AS Unique_Labels
   FROM MART_AUTHORS a
   JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
   JOIN MART_IMAGES_LABELS l ON p.IMAGE_ID = l.IMAGE_ID
   GROUP BY SEGMENT)
SELECT s1.Segment AS Segment1,
       s2.Segment AS Segment2,
       ABS(s1.Avg_Engagement_Rate - s2.Avg_Engagement_Rate) AS Engagement_Rate_Diff,
       ABS(s1.Unique_Fashion_Items - s2.Unique_Fashion_Items) AS Fashion_Item_Diff,
       ABS(s1.Unique_Labels - s2.Unique_Labels) AS Label_Diff
FROM SegmentProfiles s1
JOIN SegmentProfiles s2 ON s1.Segment < s2.Segment
ORDER BY Engagement_Rate_Diff DESC;
```
### Results:

| **Segment 1** | **Segment 2** | **Engagement Rate Diff** | **Fashion Item Diff** | **Label Diff** |
|---------------|---------------|--------------------------|-----------------------|----------------|
| Edgy          | Mainstream     | 0.0336                   | 0                     | 0              |
| Mainstream    | Trendy         | 0.0264                   | 0                     | 0              |
| Edgy          | Trendy         | 0.0072                   | 0                     | 0              |



- **Engagement Rate Diff**: This column represents the absolute difference in average engagement rates between two segments.
  - The **Edgy** and **Mainstream** segments have the largest difference in engagement rate (0.0336), indicating a significant difference in how much engagement each author type receives. **Mainstream** authors tend to have higher engagement than **Edgy** authors.
  - The difference between **Mainstream** and **Trendy** is smaller but still noticeable (0.0264).
  - **Edgy** and **Trendy** authors are the most similar in terms of engagement rates, with a very small difference (0.0072).

- **Fashion Item Diff**: This column shows the difference in the number of unique fashion items used by authors between segments.
  - All segments have a **Fashion Item Diff** of 0, indicating that the authors in all segments use a similar number of unique fashion items on average.

- **Label Diff**: This column represents the difference in the number of unique labels used by authors between segments.
  - Similarly to the fashion items, the **Label Diff** is 0 across all segment comparisons, suggesting that the number of unique labels used does not differ significantly between the segments.

### Conclusion:
The segments show some distinctiveness in terms of engagement rates, which is positive.

However, the lack of difference in Unique_Fashion_Items and Unique_Labels (all differences are 0) is concerning. This suggests that the content across segments might not be as diverse as we'd hope for a fashion-focused segmentation.


#### Step 3: Fashion-Relevance Analysis

- Assess how well each segment captures fashion-focused accounts using the FASHION_INTEREST_SEGMENT flag.



```sql
WITH FashionRelevance AS
  (SELECT CASE
              WHEN a.NB_FOLLOWERS <= 12000 THEN 'Mainstream'
              WHEN a.NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
              ELSE 'Edgy'
          END AS SEGMENT,
          AVG(CASE
                  WHEN s.FASHION_INTEREST_SEGMENT THEN 1
                  ELSE 0
              END) AS Fashion_Interest_Ratio,
          COUNT(DISTINCT l.TYPE) AS Avg_Fashion_Items_Per_Author,
          COUNT(DISTINCT l.LABEL_NAME) / COUNT(DISTINCT a.AUTHORID) AS Avg_Labels_Per_Author
   FROM MART_AUTHORS a
   JOIN MART_AUTHORS_SEGMENTATIONS s ON a.AUTHORID = s.AUTHORID
   JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
   JOIN MART_IMAGES_LABELS l ON p.IMAGE_ID = l.IMAGE_ID
   GROUP BY SEGMENT)
SELECT *
FROM FashionRelevance
ORDER BY Fashion_Interest_Ratio DESC;


```


| Segment     | Fashion_Interest_Ratio | Avg_Fashion_Items_Per_Author | Avg_Labels_Per_Author |
|-------------|------------------------|-----------------------------|-----------------------|
| Edgy        | 0.201477               | 5                           | 2044.697324           |
| Trendy      | 0.048137               | 5                           | 1191.047582           |
| Mainstream  | 0.008073               | 5                           | 817.733983            |

- **Higher Fashion_Interest_Ratio and Avg_Labels_Per_Author** suggest that a segment is more aligned with fashion-focused accounts, as authors show greater interest and variety in their fashion-related content.
- **Low scores in both metrics** indicate that the segment likely includes many accounts that are not centered around fashion, especially in the Mainstream group where fashion engagement is minimal.

### Explanation:

- **Fashion_Interest_Ratio**: This represents the proportion of authors in each segment that show an explicit interest in fashion. A higher value indicates a higher concentration of fashion-focused accounts.
  
- **Avg_Fashion_Items_Per_Author**: This column shows the average number of distinct fashion-related items that authors in each segment post about. All segments have the same number of distinct fashion items, which may suggest a similar content variety in terms of fashion-related posts across segments.

- **Avg_Labels_Per_Author**: This metric represents the average number of distinct labels associated with the posts of authors in each segment. A higher number suggests that authors in that segment are more diverse or prolific in their use of fashion-related labels, which may reflect greater activity or interest in fashion topics.

### Interpretation:

- **Edgy Segment**: With the highest **Fashion_Interest_Ratio** (0.201477) and **Avg_Labels_Per_Author** (2044.697), this segment clearly captures more fashion-focused accounts. Authors in the Edgy segment are more likely to be engaged with fashion content, which is reflected in both their higher interest and their more diverse label usage.
  
- **Trendy Segment**: This segment has a moderate **Fashion_Interest_Ratio** (0.048137), showing that some fashion-focused accounts are present, though not as concentrated as in the Edgy segment. The **Avg_Labels_Per_Author** (1191.047) is still quite high, indicating a fair degree of label diversity.

- **Mainstream Segment**: The **Fashion_Interest_Ratio** (0.008073) is the lowest, which suggests that most authors in this segment are not primarily focused on fashion. The lower **Avg_Labels_Per_Author** (817.734) also reflects this, showing less engagement with or diversity in fashion-related labels.

#### Step 4: Content Diversity Analysis:

- Analyze the variety of fashion items and labels within each segment.


### 2. What biases can you identify in the panel creation or in the accounts selection that might distort insights and how would you correct them?

Based on the analysis, I've identified the following biases:


#### Bias 1: Follower Count Overemphasis

Problem: Current segmentation relies solely on follower counts, ignoring engagement and content quality.
Possible correction : Incorporate engagement rates and content diversity into segmentation criteria.


Exemple:
```sql

WITH EngagementScores AS
  (SELECT a.AUTHORID,
          NB_FOLLOWERS,
          AVG(NB_LIKES / NULLIF(NB_FOLLOWERS, 0)) AS Avg_Engagement_Rate,
          COUNT(DISTINCT l.TYPE) AS Content_Diversity
   FROM MART_AUTHORS a
   JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
   JOIN MART_IMAGES_LABELS l ON p.IMAGE_ID = l.IMAGE_ID
   GROUP BY a.AUTHORID,
            NB_FOLLOWERS)
SELECT AUTHORID,
       CASE
           WHEN NB_FOLLOWERS > 40000
                AND Avg_Engagement_Rate > 0.1
                AND Content_Diversity > 5 THEN 'High-Impact Influencer'
           WHEN NB_FOLLOWERS BETWEEN 12001 AND 40000
                AND Avg_Engagement_Rate > 0.05 THEN 'Rising Star'
           WHEN NB_FOLLOWERS <= 12000
                AND Avg_Engagement_Rate > 0.15 THEN 'Micro-Influencer'
           ELSE 'Standard User'
       END AS Refined_Segment
FROM EngagementScores;

```

Here is an exemple, what could be the results :

| **AUTHORID**      | **Refined Segment**      |
|-------------------|--------------------------|
| XXXX56037         | Standard User            |
| XXXX5776          | High-Impact Influencer            |
| XXXX1072381       | Rising Star            |
| XXXX6911574       | Micro-Influencer         |
| XXXX28055         | Standard User            |


- This query corrects for follower count bias by adding **engagement rate** and **content diversity** as segmentation factors.
- High-impact influencers are now identified based on **engagement rate** and **diversity** of content, not just follower count.
- **Micro-influencers** with small but highly engaged followings are recognized, balancing follower numbers with meaningful interactions.


- **Follower count alone** is not sufficient to define an account's value. High follower counts without engagement or diverse content may skew insights.
- By adding **engagement rates** and **content variety**, the segmentation captures **true influence**, helping to distinguish between **authentic influencers** and **standard users** with less impact.



#### Bias 2: Temporal Bias

The current segmentation ignores account activity patterns and post frequency, leading to a potential bias in identifying relevant influencers. Without considering how often and recently users post, influencers who are inactive but have high follower counts may still appear as valuable, distorting insights.

Analysis:
```

WITH AuthorActivity AS
  (SELECT a.AUTHORID,
          MIN(p.POST_PUBLICATION_DATE) AS First_Post,
          MAX(p.POST_PUBLICATION_DATE) AS Last_Post,
          COUNT(*) AS Post_Count,
          DATEDIFF('day', MIN(p.POST_PUBLICATION_DATE), MAX(p.POST_PUBLICATION_DATE)) AS Active_Days
   FROM MART_AUTHORS a
   JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
   GROUP BY a.AUTHORID)
SELECT CASE
           WHEN NB_FOLLOWERS <= 12000 THEN 'Mainstream'
           WHEN NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
           ELSE 'Edgy'
       END AS SEGMENT,
       AVG(Post_Count / NULLIF(Active_Days, 0)) AS Avg_Posts_Per_Day,
       AVG(DATEDIFF('day', Last_Post, CURRENT_DATE)) AS Avg_Days_Since_Last_Post
FROM AuthorActivity
JOIN MART_AUTHORS USING (AUTHORID)
GROUP BY SEGMENT;

```

| **Segment**  | **Avg Posts Per Day** | **Avg Days Since Last Post** |
|--------------|-----------------------|------------------------------|
| **Trendy**   | 0.2588                | 173.11                       |
| **Mainstream** | 0.2173              | 174.60                       |
| **Edgy**     | 0.3436                | 174.38                       |

- **Avg Posts Per Day**: Influencers in the **Edgy** segment post more frequently, while **Mainstream** and **Trendy** influencers post less frequently.
- **Avg Days Since Last Post**: All segments show similar levels of inactivity, with around 173–174 days since their last post on average.

### **Conclusion**:
- **Incorporate activity metrics**: To better evaluate influencer relevance, segmentation should include metrics for post frequency and recency, ensuring active influencers are prioritized over inactive ones.
- **Time decay factor**: Adjust engagement metrics to give more weight to recent posts, correcting for the overemphasis on static follower counts.

This approach will better reflect the true influence and activity of users, reducing the bias introduced by ignoring their posting patterns and timelines.

b) Lack of Content and Style Differentiation:

- Analysis shows no difference in unique fashion items or labels across segments.
- Correction: Include content-based metrics in segmentation, such as the types of fashion items featured.

c) Engagement Rate Variability:

- The Mainstream segment shows high variability in engagement rates.
- Correction: Consider subdividing this segment or using engagement rate thresholds in addition to follower counts.

d) Fashion Relevance Misalignment:

- Segments may not accurately reflect fashion-forward accounts.
- Correction: Integrate the FASHION_INTEREST_SEGMENT flag into the segmentation process.

e) Potential Geographic Imbalance:

- Certain regions might be over or underrepresented in the panel.
- Correction: Analyze geographic distribution and consider implementing geographic quotas or weighting.


## To correct these biases:

This is what propose :

### Refine Segmentation Criteria:
- Combining follower count, engagement, and fashion interest for a better view
- Use **engagement rates** and **content diversity** alongside follower counts.
- Incorporate the `FASHION_INTEREST_SEGMENT` flag to prioritize fashion-forward accounts.

### Content-Based Differentiation:
- Incorporating content diversity to ensure segments reflect varied fashion content
- Creating more nuanced segments that better reflect an author's true influence in the fashion space
- Focusing on analyzing the most frequent **fashion items** and **labels** per segment.
- Use this information to tailor **marketing campaigns** for each segment.

### Geographic Analysis:
- Include other regional data other than **US**.
- Implement **weighting factors** or **quotas** for underrepresented regions.

