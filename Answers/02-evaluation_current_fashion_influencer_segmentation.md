# Evaluation of Current Fashion Influencer Segmentation

## The Question

What do you think about the current segmentation? What advantages / drawbacks do you see in the methodology / in the way the information is stored? You can play around with the provided data to support your statements.

## What We Understood

1. Current Segmentation:
    - Based solely on follower counts:
        - Mainstream: ≤12,000 followers
        - Trendy: 12,001-40,000 followers
        - Edgy: >40,000 followers
    - Stored in `MART_AUTHORS_SEGMENTATIONS` table
2. Available Data:
    - `MART_AUTHORS`: Author metadata
    - `MART_AUTHORS_SEGMENTATIONS`: Current segmentation data, includes FASHION_INTEREST_SEGMENT
    - `MART_IMAGES_LABELS`: Image analysis results (TYPE, POSITION_IN_IMAGE, LABEL_NAME)
    - `MART_IMAGES_OF_POSTS`: Links posts to images
3. Task: Evaluate the current segmentation methodology and data storage approach

## Analysis and Results

### 1. Segmentation Distribution Analysis

Let's first look at the distribution of authors across segments:

```sql
SELECT 
  CASE 
    WHEN NB_FOLLOWERS <= 12000 THEN 'Mainstream'
    WHEN NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
    ELSE 'Edgy'
  END AS Segment,
  COUNT(*) AS Author_Count,
  AVG(NB_FOLLOWERS) AS Avg_Followers
FROM MART_AUTHORS
GROUP BY Segment
ORDER BY Avg_Followers;
```

Results:


| Segment | Author_Count | Avg_Followers |
| :-- | :-- | :-- |
| Mainstream | 19588 | 1781.669389 |
| Trendy | 1397 | 20978.509664 |
| Edgy | 1454 | 870214.255856 |


Interpretation:
Mainstream Segment: The vast majority of authors are in the "Mainstream" segment, with a relatively low average follower count (~1,782). This segment likely represents everyday users or micro-influencers with smaller but possibly more engaged audiences.

Trendy Segment: Authors in the "Trendy" segment have significantly more followers on average (~20,979), suggesting they are emerging influencers or niche trendsetters with a growing presence in the fashion community.

Edgy Segment: The "Edgy" segment has the fewest authors but the highest average follower count (~870,214). This segment represents established, high-impact influencers with substantial reach and authority in the fashion world.

Key Insight: There’s a large disparity between the "Mainstream" and "Edgy" segments, both in terms of the number of authors and average followers, indicating different levels of influence and reach across these groups. The segmentation can help target marketing efforts according to the scale of influence.





### 2. Engagement Rate Analysis

Let's compare engagement rates across segments:

```sql
WITH SegmentEngagement AS (
  SELECT 
    CASE 
      WHEN a.NB_FOLLOWERS <= 12000 THEN 'Mainstream'
      WHEN a.NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
      ELSE 'Edgy'
    END AS Segment,
    AVG(p.NB_LIKES / NULLIF(a.NB_FOLLOWERS, 0)) AS Avg_Engagement_Rate
  FROM MART_AUTHORS a
  JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
  GROUP BY Segment
)
SELECT 
  Segment,
  Avg_Engagement_Rate,
  RANK() OVER (ORDER BY Avg_Engagement_Rate DESC) AS Engagement_Rank
FROM SegmentEngagement;
```

Results:


| Segment | Avg_Engagement_Rate | Engagement_Rank |
| :-- | :-- | :-- |
| Mainstream |	0.038889320027 |	1 |
| Trendy |	0.020913803077 |	2 |
| Edgy	| 0.013263590449 |	3 |


Interpretation:

### Interpretation:

- **Mainstream Segment:** Authors in the "Mainstream" segment have the highest average engagement rate (~3.89%). Despite having fewer followers on average, they are likely to have closer and more personal connections with their audience, which leads to higher engagement.
  
- **Trendy Segment:** The "Trendy" segment, with an average engagement rate of ~2.09%, still shows decent interaction, though engagement begins to drop as follower count increases. These influencers may be in a transitional phase where they are growing their audience but not yet maintaining the same level of personal interaction as the "Mainstream" group.
  
- **Edgy Segment:** The "Edgy" segment has the lowest average engagement rate (~1.33%). These large influencers may have a more passive audience, or their higher follower count could dilute the engagement. Despite their reach, their interaction with individual posts might be lower compared to the smaller segments.

- **Key Insight:** While the "Edgy" segment has more followers, the engagement rate tends to decrease as audience size increases, highlighting the importance of targeting micro and emerging influencers ("Mainstream" and "Trendy") for campaigns focused on engagement rather than just reach.

### 3. Fashion Interest Analysis

Let's examine how the FASHION_INTEREST_SEGMENT flag is distributed:

```sql
SELECT 
  CASE 
    WHEN a.NB_FOLLOWERS <= 12000 THEN 'Mainstream'
    WHEN a.NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
    ELSE 'Edgy'
  END AS Segment,
  AVG(CASE WHEN s.FASHION_INTEREST_SEGMENT THEN 1 ELSE 0 END) AS Fashion_Interest_Ratio
FROM MART_AUTHORS a
JOIN MART_AUTHORS_SEGMENTATIONS s ON a.AUTHORID = s.AUTHORID
GROUP BY Segment
ORDER BY Fashion_Interest_Ratio DESC;
```

Results:


| Segment | Fashion_Interest_Ratio |
| :-- | :-- |
| Edgy | 0.083219 |
| Trendy | 0.022190 |
| Mainstream | 0.006841 |


### Interpretation:

- **Edgy Segment:** Authors in the "Edgy" segment have the highest proportion of accounts flagged for fashion interest (~8.32%). This suggests that larger influencers are more likely to be fashion-focused, possibly due to their association with fashion brands or their reach in fashion-forward communities.
  
- **Trendy Segment:** The "Trendy" segment shows a moderate fashion interest ratio (~2.22%), indicating that some emerging influencers are also focused on fashion, but less consistently compared to the "Edgy" group.
  
- **Mainstream Segment:** The "Mainstream" segment has the lowest fashion interest ratio (~0.68%), suggesting that while they may have high engagement, they are less likely to be focused on fashion.


Edgy influencers tend to have a stronger alignment with fashion interests, whereas smaller influencers ("Mainstream") may be less fashion-focused but more engaged with their audience. For fashion-focused marketing campaigns, prioritizing "Edgy" influencers with high fashion interest could be more effective.

### 4. Content Diversity Analysis

The Content Diversity Analysis provides insights into how diverse the fashion content is across different segments of influencers.


```sql
SELECT 
  CASE 
    WHEN a.NB_FOLLOWERS <= 12000 THEN 'Mainstream'
    WHEN a.NB_FOLLOWERS BETWEEN 12001 AND 40000 THEN 'Trendy'
    ELSE 'Edgy'
  END AS Segment,
  COUNT(DISTINCT l.TYPE) AS Unique_Fashion_Items,
  COUNT( l.LABEL_NAME) AS Unique_Labels
FROM MART_AUTHORS a
JOIN MART_IMAGES_OF_POSTS p ON a.AUTHORID = p.AUTHORID
JOIN MART_IMAGES_LABELS l ON p.IMAGE_ID = l.IMAGE_ID
GROUP BY Segment;
```

Results:


| Segment | Unique_Fashion_Items | Unique_Labels |
| :-- | :-- | :-- |
| Mainstream | 5 | 13771458 |
| Trendy | 5 | 1526923 |
| Edgy | 5 | 2445458 |



### Content Diversity Analysis Interpretation

- **Fashion Items**: All segments (Mainstream, Trendy, Edgy) engage with the same number of unique fashion items (5), indicating no major difference in item variety across segments.
  
- **Label Diversity**: 
   - **Mainstream influencers** cover the most unique labels (13.7M), reflecting their broad appeal.
   - **Trendy influencers** focus on fewer labels (1.5M), suggesting a more niche approach.
   - **Edgy influencers** engage with 2.4M labels, balancing between mainstream and niche.

Mainstream influencers have the widest brand engagement, while Trendy influencers lean towards more specific labels. 
Marketing strategies should reflect these distinctions in targeting.

## Advantages and Drawbacks

---

## Advantages and Drawbacks

### Advantages:

1. **Simplicity:**
   - The segmentation is straightforward and easy to understand, based solely on follower counts.

2. **Scalability:**
   - Follower count is a readily available metric that can be updated easily as authors grow their audiences.

3. **Clear Boundaries:**
   - The segmentation provides clear cut-off points for each category, simplifying implementation.

4. **Marketing Alignment:**
   - Segments align well with marketing strategies targeting reach (Edgy), growth (Trendy), or engagement (Mainstream).

---

### Drawbacks:

1. **Oversimplification:**
   - Relying solely on follower count ignores other important factors like engagement rate, content quality, and fashion relevance.

2. **Engagement Misalignment:**
   - Larger influencers ("Edgy") have lower engagement rates compared to smaller ones ("Mainstream"), which may lead to suboptimal targeting for engagement-focused campaigns.

3. **Fashion Interest Misalignment:**
   - The "Mainstream" segment has high engagement but low alignment with fashion interests, making it less suitable for fashion-specific campaigns.

4. **Lack of Content Differentiation:**
   - All segments engage with the same number of unique fashion items, showing little variation in content focus across segments.

5. **Static Nature:**
   - The current methodology does not account for changes in an author's influence or behavior over time.

6. **Storage Limitations:**
   - While storing segmentation data separately is flexible, it lacks granularity for more nuanced segmentation criteria like engagement trends or content preferences.

---

## Recommendations

### Methodology Improvements:

1. **Multi-Factor Segmentation:**
   Incorporate additional metrics such as engagement rate, content diversity, and `FASHION_INTEREST_SEGMENT` into segmentation criteria alongside follower counts.

2. **Dynamic Segmentation:**
   Regularly update segments based on recent performance metrics (e.g., engagement trends) to ensure relevance over time.

3. **Content-Based Classification:**
   Use data from `MART_IMAGES_LABELS` to create more meaningful segments based on the types of fashion items and labels featured by authors.

4. **Engagement Quality Metrics:**
   Include metrics like comment-to-like ratio or sentiment analysis of comments to better understand audience interaction quality.

5. **Temporal Tracking ():**
   Add timestamp fields to track when an author entered or left a segment and analyze how their behavior evolves over time.

6. **Geographic Analysis ():**
   Incorporate geographic data (`GEO_ZONE`) to identify regional imbalances and implement weighting factors or quotas for underrepresented regions.

---

### Storage Improvements:

1. Add fields for dynamic metrics:
   - Engagement trends (e.g., average engagement over time).
   - Content diversity scores (e.g., number of unique labels).

2. Implement historical tracking:
   - Store snapshots of segmentation criteria at regular intervals to analyze changes over time.

---

By addressing these drawbacks and implementing these recommendations, we can create a more nuanced and effective segmentation strategy that better captures the complexities of fashion influence on social media while improving data storage flexibility for future analyses.

---
Réponse de Perplexity: pplx.ai/share