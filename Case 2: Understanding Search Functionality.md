# Context
The product team is determining priorities for the next development cycle and they are considering improving the site's search functionality. It currently works as follows:

* There is a search box in the header the persists on every page of the website. It prompts users to search for people, groups, and conversations.
![image](https://user-images.githubusercontent.com/32259752/174474561-b97c488b-315a-41db-8952-40c8e5901e99.png)

* When a user begins to type in the search box, a dropdown list with the most relevant results appears. The results are separated by category (people, conversations, files, etc.). There is also an option to view all results.
![image](https://user-images.githubusercontent.com/32259752/174474585-0328195a-9873-4de6-ab8c-64d56bcb7fcf.png)

* When the user hits enter or selects “view all results” from the dropdown, she is taken to a results page, with results separated by tabs for different categories (people, conversations, etc.). Each tab is order by relevance and chronology (more recent posts surface higher).
![image](https://user-images.githubusercontent.com/32259752/174474643-e21cd6cc-f626-4a82-8b5f-4996eebe2b86.png)

* The search results page also has an “advanced search” box that allows the user to search again within a specific Yammer group or date range.



# Problem
Before tackling search, the product team wants to make sure that the engineering team's time will be well-spent in doing so. So, I want to know:
1. Are users' search experiences generally good or bad?
2. Is search worth working on at all?
3. If search is worth working on, what, specifically, should be improved?
4. If the recommendations would be completed, how to understand the recommendations are actually improvements of the old search function?


# Question 1: Are users' search experiences generally good or bad?
First, I need to define success metrics to undertand if the search function is satisfactory for now to our users. 
For the search function, three metrics matter the most:

1. **Retention Rate**: percentage of users continuing using the function
2. **Churn Rate**: percentage of usrs turning away from the function
3. **Average Session Duration**: period of time in which a user starts the search until their last action in a search session.

## Retention Rate and Churn Rate
In this query, `engaged_user_count` is defined as the number of users who 
used the search function at least once in the month. `retention_count` is defined as the number of users who used the function last month and continue to use the 
function this month. `retention_rate` is defined as the percentage of the retained users of all the engaged users this month. 
`churn_count` is defined as the number of users who used the function last month but not this month. `churn_rate` is defined as the percentage of churned 
users of all the engaged users last month.

According to [an article from Jenny Booth](https://mixpanel.com/blog/whats-a-good-retention-rate/), 
Marketing Manager @ Mixpanel, "For the SaaS and e-commerce industries, **over 35 percent retention is considered elite**." You can see since the launch 
of the search function, the retention rate fluctuated betwwen 30% to 36%, which is a decent number for Yammer.

But the churn rate is a red flag. According to [Patrick Campbell](https://twitter.com/patticus?lang=en), the founder and CEO of ProfitWell, 
for B2C SaaS: **Between 3% and 5% monthly churn is GOOD, and less than 2% is GREAT.** In the perspective, the search function is far from being good, though 
this high churn rate could be accounted by seasonalilty to some extent. Yammer is a business service. Some users take summer vacation in July and August. It makes sense 
that some users quitted using the search function in the summer months.

```
WITH monthly_activity AS (
  SELECT distinct DATE_TRUNC('month', occurred_at) AS month,
  user_id
FROM tutorial.yammer_events
WHERE event_name LIKE 'search%'
)

, monthly_engaged_user_count AS (
SELECT month, COUNT(DISTINCT user_id) AS engaged_user_count
FROM monthly_activity
GROUP BY 1
)

, monthly_retention_count AS (
SELECT this_month.month AS month, COUNT(DISTINCT this_month.user_id) AS retention_count
FROM monthly_activity this_month
JOIN monthly_activity last_month
ON this_month.user_id = last_month.user_id
AND this_month.month = last_month.month + INTERVAL '1 month'
GROUP BY 1
)

, monthly_churn_count AS (
SELECT this_month.month+ INTERVAL '1 month' AS month, COUNT(DISTINCT last_month.user_id) AS churn_count
FROM monthly_activity this_month
LEFT JOIN monthly_activity last_month
ON this_month.user_id = last_month.user_id
AND this_month.month = last_month.month + '1 month'
GROUP BY 1
)

SELECT 
  u.month, 
  u.engaged_user_count,
  r.retention_count, 
  round(r.retention_count*1.00/u.engaged_user_count*100,2) AS retention_rate,
  c.churn_count,
  ROUND((c.churn_count*1.00/LAG(u.engaged_user_count) OVER (ORDER BY u.month))*100, 2) AS churn_rate
FROM monthly_engaged_user_count u
JOIN monthly_retention_count r
ON u.month = r.month
JOIN monthly_churn_count c 
ON c.month = u.month
![image](https://user-images.githubusercontent.com/32259752/175207403-6c2903d6-95c5-4ad6-afb1-b3419f171524.png)
```

<img width="468" alt="image" src="https://user-images.githubusercontent.com/32259752/175207418-8017950c-f85c-4285-aa09-c4d9fae80e3e.png">

## Average Session Duration
What is important of a search function is how long it takes an user to find the result. `average_session_duration` is defined as the average time 
of each search session, which is the time span in which an user starts with a search-related event to they finish with another search-related action. 
The whole session should be in 10 minutes. If the session is longer than 10 minutes, I would call it another session.
The shorter the `average_session_duration` is, the less time an user spend on each search.

On average, it took an user about 3 minutes and 13 seconds to finish a search session. If I put myself in the context, this is too long.
It means the search function is not efficient enough for now.

```
SELECT AVG(session_start_end.session_end - session_start_end.session_start) AS avg_session_duration
FROM      ( SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2) session_start_end
```

<img width="221" alt="image" src="https://user-images.githubusercontent.com/32259752/175815170-b25872a9-d71a-4cb5-8d4a-ba0118ac526c.png">


## Summary
Although the search function had a high retention rate, many users churned away from it because it took too long for them to find the result.

# Question 2. Is search worth working on at all?
The answer for me: YES.

Yammer is a social network for communicating with coworkers in sharing documents, updates, and ideas by posting them in groups. 
There would be a huge media library stored in the server for every user.  Without a convenient search function, it would be impossible for uesrs to elegantly 
find the resource they want. 

Though the user experience of the search function is not satisfactory yet (based on `average_search_duration` and `churn_rate`), 
it has a solid user foundation (based on `retention_rate`).

In the long run, the search function helps users to communicate with coworkers efficiently, bringing more potential users to the platform.
This in turn would result in more revenue for the company. The effort and time would outweigh the drawbacks.

# Question 3. If search is worth working on, what, specifically, should be improved?

## Increase user awareness of the search function
Users who used the search function accounted for about 18% to 22% of all the active users of Yammer service, which is not very high.
It is possible that some users simply did not realize the existence of the function, or they did not understand how to use it.
There should be a tutorial for first-time users of the search function, showing them how to use it step by step.

```
WITH active_user AS (SELECT DATE_TRUNC('month', created_at) AS month,
      COUNT(DISTINCT user_id) AS active_user_count
FROM tutorial.yammer_users
WHERE state='active'
GROUP BY month)

, search_user AS (
SELECT DATE_TRUNC('month', occurred_at) AS month,
      COUNT(DISTINCT user_id) AS search_user_count
FROM tutorial.yammer_events 

WHERE event_name LIKE 'search%'
GROUP BY month
)

SELECT 
  active.month, 
  SUM(active.active_user_count) OVER (ORDER BY active.month) AS running_total_active_user,
  search.search_user_count,
  search.search_user_count*1.00/SUM(active.active_user_count) OVER (ORDER BY active.month)*100 AS percentage
FROM active_user active
LEFT JOIN search_user search 
ON active.month = search.month
```

<img width="468" alt="image" src="https://user-images.githubusercontent.com/32259752/175233114-72456061-54fa-4ec1-8527-1f7a9b801f81.png">

## Improve the autocomplete algorithm

For users, autocomplete should be most troule-free way to get the result. A user can go for an intended 
result staying in the current page they are on. But in the query, you can see that users did not use autocomplete that much (about 57%). This means 
the enginnering team should work on the autocomplte function, helping users get the result with least friction.

```
WITH sub AS (SELECT 
  SUM(CASE WHEN event_name = 'search_autocomplete' THEN 1 ELSE 0 END) AS autocomplete_count,
  SUM(CASE WHEN event_name = 'search_run' THEN 1 ELSE 0 END) AS search_run_count
FROM tutorial.yammer_events)

SELECT 100*(autocomplete_count)/(autocomplete_count + search_run_count) AS autocomplete_percentage
FROM sub
```

<img width="230" alt="image" src="https://user-images.githubusercontent.com/32259752/175820764-03d210cd-c8fe-4de8-ae8e-ccbba502da31.png">


## Improve the search ranking algorithm
To redesign the search result ranking system is a good idea. If users can find what they are looking for without scrolling the result page, 
they would be more likely to use the search function.

The search result is ranked based on the relevance and chronology. So, users should click on search_click_result 1 most often. But now users clicked on 
result 2 more often than result 1, result 4 more than result 3, result 9 more than result 7 and 8.

```
SELECT event_name, COUNT(event_name)
from tutorial.yammer_events
WHERE event_name LIKE 'search_click_%'
GROUP BY 1
ORDER BY 2 DESC
```

<img width="310" alt="image" src="https://user-images.githubusercontent.com/32259752/175233994-eb740412-d11a-4e06-8aac-7d4a12ebb772.png">

## Improve language localization 
The customer experience of the search function vary in different languages because localization could be a challenge for search 
function enginnering team. I want to study users by languages.

From the query, I can see that we lost most search users in English, French, and Spainish. So, we can improve the function by providing better 
search experiences for these language users.

```
WITH language_segment AS (
SELECT 
  DATE_TRUNC('month', e.occurred_at) AS month,
  u.language, 
  COUNT(DISTINCT e.user_id)
FROM tutorial.yammer_users u
JOIN tutorial.yammer_events e 
ON u.user_id = e.user_id 
WHERE event_name LIKE 'search%'
GROUP BY 2,1
)

SELECT 
  month,
  language,
  count,
  count - LAG(count) OVER (PARTITION BY language ORDER BY month) AS customer_increase
FROM language_segment
ORDER BY 4
```

<img width="468" alt="image" src="https://user-images.githubusercontent.com/32259752/175316278-15fb0f77-34c1-4d64-a040-f14cd6777aa4.png">

## Summary
For the improvement of the search function, I recommend:
1. Improve the autocomplete algorithm
2. Improve the search ranking algorithm
3. Improve language localization 

I would suggest working on recommendation 1 first. Of limited engineering resource, we need to prioritize which recommendation has the most impact. 
For a search function, the smoother users can find the result, the more willing they are to use the function. 
For this aspect, I would say autocomplete is the most convenient way to 
access the result. The improvement of this function would greatly increase the confidence of the users in the search function.

# Question 4: If the recommendations will be completed, how to understand the recommendations are actually improvements of the old search function?

The improvement metrics are the same as the success metrics of the function:
1. Retention rate
2. Churn rate
3. Average session duration

If more users keep using the function, less users turning away from it, and the average search session is getting shorter, then I know the search function is 
making progress over the old search function.






