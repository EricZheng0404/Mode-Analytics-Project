# 1. Context
Yammer not only develops new features, but is continuously looking for ways to improving existing ones. 
Like many software companies, Yammer frequently tests these features before releasing them to all of thier customers. 
These A/B tests help analysts and product managers better understand a feature's effect on user behavior and the overall user experience.

This case focuses on an improvement to Yammer's core “publisher”—the module at the top of a Yammer feed where users type their messages. 
To test this feature, the product team ran an A/B test from June 1 through June 30. During this period, some users who logged into Yammer were shown the old version of the publisher (the “control group”), while other other users were shown the new version (the “treatment group”).

![image](https://user-images.githubusercontent.com/32259752/177293130-83217a2f-ab9a-43af-add2-b08fd4fb8309.png)

# 2. Problem
On July 1, you check the results of the A/B test. You notice that message posting is 50% higher in the treatment group—a huge increase in posting. 
The table below summarizes the results:

<img width="1300" alt="image" src="https://user-images.githubusercontent.com/32259752/177293411-f326657c-7887-42ea-a13b-ebf6cbb621d7.png">

The chart shows the average number of messages posted per user by treatment group. The table below provides additional test result details:

* **users**: The total number of users shown that version of the publisher.
* **totaltreatedusers**: The number of users who were treated in either group.
* **treatment_percent**: The number of users in that group as a percentage of the total number of treated users.
* **total**: The total number of messages posted by that treatment group.
* **average**: The average number of messages per user in that treatment group (total/users).
* **rate_difference**: The difference in posting rates between treatment groups (group average - control group average).
* **rate_lift**: The percent difference in posting rates between treatment groups ((group average / control group average) - 1).
* **stdev**: The standard deviation of messages posted per user for users in the treatment group. For example, if there were three people in the control group and they posted 1, 4, and 8 messages, this value would be the standard deviation of 1, 4, and 8 (which is 2.9).
* **t_stat**: A test statistic for calculating if average of the treatment group is statistically different from the average of the control group. It is calculated using the averages and standard deviations of the treatment and control groups.
* **p_value**: Used to determine the test's statistical significance.

The problem is: whether this feature is the real deal or too good to be true? I need to make a final recommendation based on your conclusions. Should the new publisher be rolled out to everyone? Should it be re-tested? If so, what should be different? Should it be abandoned entirely?

# 3. The data
## Table 1: Users

|Column name| Description |
| --- | --- |
| user_id |	A unique ID per user. Can be joined to user_id in either of the other tables. |
|created_at|The time the user was created (first signed up)|
|state|	The state of the user (active or pending)|
|activated_at|	The time the user was activated, if they are active|
|company_id|	The ID of the user's company|
|language|	The chosen language of the user|

## Table 2: Events
|Column name| Description |
| --- | --- |
|user_id|	The ID of the user logging the event. Can be joined to user\_id in either of the other tables.|
|occurred_at|	The time the event occurred.|
|event_type|	The general event type. There are two values in this dataset: "signup_flow", which refers to anything occuring during the process of a user's authentication, and "engagement", which refers to general product usage after the user has signed up for the first time.|
|event_name|	The specific action the user took. Possible values include: create_user: User is added to Yammer's database during signup process enter_email: User begins the signup process by entering her email address enter_info: User enters her name and personal information during signup process complete_signup: User completes the entire signup/authentication process home_page: User loads the home page like_message: User likes another user's message login: User logs into Yammer search_autocomplete: User selects a search result from the autocomplete list search_run: User runs a search query and is taken to the search results page search_click_result_X: User clicks search result X on the results page, where X is a number from 1 through 10. send_message: User posts a message view_inbox: User views messages in her inbox|
|location|	The country from which the event was logged (collected through IP address).|
|device|	The type of device used to log the event.|

## Table 3: Experiments
|Column name| Description |
| --- | --- |
|user_id:	The ID of the user logging the event. Can be joined to user_id in either of the other tables.|
|occurred_at|	The time the user was treated in that particular group.|
|experiment|	The name of the experiment. This indicates what actually changed in the product during the experiment.|
|experiment_group|	The group into which the user was sorted. "test_group" is the new version of the feature; "control_group" is the old version.|
|location|	The country in which the user was located when sorted into a group (collected through IP address).|
|device|	The type of device used to log the event.|

## Table 4: Normal Distribution
|Column name| Description |
| --- | --- |
|score|	Z-score. Note that this table only contains values >= 0, so you will need to join the absolute value of the Z-score against it.|
|value|	The area on a normal distribution below the Z-Score.|

# 4. Hypotheses
According to **Twyman's Law**, any statistics that appears interesting is almost certainly a mistake. In this case, we can see that t-stat is 7.6245, 
p-value is 0, which means we are almost 100% confident that the treatment was having effect on the users. The statistics numbers were very extreme. So,
I want to check:
1. **Internal Validity**. If the experimental results are correct without attempting to generalize to other populations or time periods.
* Did the outliers of average number of messages per user have an huge impact of the result?
* Is the average number of messages per user a reasonable success metric? While it was rising, did it degrade other success and guardrail metrics?
* Is the hypothesis testing correct?
* Is the rise in the average number of messages per user a surviorship bias? Is it possible that those who published before already published more, but those who did not publish still did not publi.

2. **External validity**. If the experimental results are valid, can they be generalized?
* Is this rise in the average number of messages per user a novelty effect? Users used it much more than before simply because it was a new feature, 
rather than because they needed it.
* Can we observe the rise in all different countries?
* Can we observe the rise in all different devices?
* Can we observe the rise in users with different activation dates?
* Can we observe the rise in users in different languages?

# 5. Testing
## 1. Internal validity
### Did the outliers of average number of messages per user have an huge impact of the result?
When such an extreme T-score came up, I wonder if outliers could have a large impact on the result. So, I want to know if the difference between 
treatment and control group will still be significant if I exclude some edge data.

```
select 
  count(distinct user_id) as treatment_count_user,
  sum(count_messages) as sum_messages,
  sum(count_messages)*1.00/count(distinct user_id) messages_per_user,
  stddev_samp(count_messages)
from (select 
      *, ntile(100) OVER (ORDER BY count_messages) as percentile
      from (select 
         ex.user_id,
         count(e.event_name) as count_messages
        from tutorial.yammer_experiments ex 
        LEFT JOIN tutorial.yammer_events e 
        on ex.user_id = e.user_id 
        and e.occurred_at >= '2014-06-01'
        and e.occurred_at < '2014-07-01'
        and e.event_name = 'send_message'
        where ex.experiment_group = 'test_group'
        group by 1
        order by 2 desc) count
    order by percentile desc) percentile
where percentile.percentile >5 and percentile.percentile <95
```

<img width="607" alt="image" src="https://user-images.githubusercontent.com/32259752/178396152-3a83912f-5ea9-4e4d-bef9-c6294394b853.png">

In the query, I remove top 5% and bottom 5% of users in terms of the number of the messages they send. I set up an one-tailed test: the Null hypothesis 
is there is no difference betwwen treatment and control group in average number of messages per user, and the alternative hypothesis is that 
average number of messages per user for treatment group is greater than that in control group. The T-score is -5.227 and P-value is 0. The
alternative hypothesis is still significant even if I remove the outliers. It means the treatment group had a greater average number of messages sent than 
the control group.

### Is the average number of messages per user a reasonable success metric? While it was rising, did it degrade other success and guardrail metrics?
I would say the average number of messages per user is a reasonable success metric. For Yammer, a social networking company, the 
engagement of users should be a Overall Evaluation Criterion because it drives the long-term objective (revenue) of the company. 
In the context of testing the new Publisher feature, the average number of messages per user is a measurable metric for engagement. It is also simple to 
calculate and easy to understand.

But other success metrics are equally important. For example, the average number of user daily login is also an important success metric for 
user engagement. When the number is high, it means users log in Yammer more frequently, which indicates their level of engagement is high.

```
select 
  case when e.occurred_at >='2014-06-02' and e.occurred_at < '2014-06-09' then 'week_1'
       when e.occurred_at >='2014-06-09' and e.occurred_at < '2014-06-16' then 'week_2'
       when e.occurred_at >='2014-06-16' and e.occurred_at < '2014-06-23' then 'week_3'
       when e.occurred_at >='2014-06-23' and e.occurred_at < '2014-06-30' then 'week_4'  end as week,
  count(case when ex.experiment_group = 'test_group' then 1 else null end)*1.00/849 as treatment_group_login,
  count(case when ex.experiment_group = 'control_group' then 1 else null end)*1.00/1746 as control_group_login
from tutorial.yammer_experiments ex 
LEFT JOIN tutorial.yammer_events e 
on ex.user_id = e.user_id 
and e.occurred_at >= ex.occurred_at
and e.occurred_at < '2014-07-01'
and e.event_name = 'login'
group by 1
order by 1
```

<img width="511" alt="image" src="https://user-images.githubusercontent.com/32259752/178406412-d3143af9-274c-4ab3-ac88-dc9b9485ba39.png">

In the query, we can see throughout four weeks of the experiment, users in treatment group logged in Yammer more often than those in control group. So, the 
new version of Publisher not only prompt users to publish more messages, it also made users more engaged in Yammer itself.

Now I want to test even if the the average number of messages per user was rising, did it degrade guardrails metrics, such as engagement of other 
Yammer events. I will 
use the average number of engagements (send_message not included) per user as an indicator to compare control and treatment group.

```
with treatment as (
select sum(count_event)/count(user_id) as average_engagements_per_user_treatment
from (select 
  distinct ex.user_id,
  count(event_name) as count_event
from tutorial.yammer_experiments ex 
LEFT JOIN tutorial.yammer_events e 
on ex.user_id = e.user_id 
and e.event_name <> 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'test_group'
group by 1) x)

, control as (
select sum(count_event)/count(user_id) as average_engagements_per_user_control
from (select 
  distinct ex.user_id,
  count(event_name) as count_event
from tutorial.yammer_experiments ex 
LEFT JOIN tutorial.yammer_events e 
on ex.user_id = e.user_id 
and e.event_name <> 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'control_group'
group by 1) x
)

select treatment.average_engagements_per_user_treatment, control.average_engagements_per_user_control
from treatment, control
```

<img width="588" alt="image" src="https://user-images.githubusercontent.com/32259752/178128445-e052b078-1422-4893-bd7b-5d05ea240818.png">

You can see even when `send_message` is not included in the `event_name`, the average number of engagements in treatement group is 
still higher than that in control group. This means with the average number of messages per user increases, no tradeoff is made.

```
with treatment as (
select 
  distinct e.event_name,
  count(e.event_name)*1.00/count(distinct ex.user_id) as average_engagements_per_user_treatment
from tutorial.yammer_experiments ex 
LEFT JOIN tutorial.yammer_events e 
on ex.user_id = e.user_id 
and e.event_name <> 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'test_group'
group by 1)

,control as (
select 
  distinct e.event_name,
  count(e.event_name)*1.00/count(distinct ex.user_id) as average_engagements_per_user_control
from tutorial.yammer_experiments ex 
LEFT JOIN tutorial.yammer_events e 
on ex.user_id = e.user_id 
and e.event_name <> 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'control_group'
group by 1)

select 
  treatment.event_name, treatment.average_engagements_per_user_treatment, control.average_engagements_per_user_control,
  (treatment.average_engagements_per_user_treatment - control.average_engagements_per_user_control) as difference
from treatment
join control
on treatment.event_name = control.event_name
order by 4 desc
```

<img width="830" alt="image" src="https://user-images.githubusercontent.com/32259752/178129162-64f9bc53-8e22-496e-81b5-6c96af53057c.png">

To break down the number, we can see that with the introduction of the new feature, `home_page`, `like_message`, and `view_inbox` also increased. There was 
no significant increase in other kinds of engagement, which is a good sign. Engagements such as `search_click_result` could serve as invariant metrics, 
these metrics should not change between the control and treatment. It means the measured difference is due to the introduction of the new feature.

## Is it possible we overfocus on the success of the result that those who published before the experiment already published more, but those who did not publish still did not publish.
Survivorship bias is the tendency to overly focus on the performance of treatment users in the experiment as a representative comprehensive sample 
without regarding those who did not use the new feature even before the introduction of the new feature. 
So, I want to know if uesrs in control group were less engaged in posting messages than those in treatment group even before then introduction of 
the new feature.

```
SELECT 
count(user_id),
avg(count_engagement) as avg_message,
stddev_samp(count_engagement) as std_message
FROM (SELECT 
        distinct ex.user_id,
        COUNT(e.event_name) AS count_engagement
      from tutorial.yammer_experiments ex 
      left join tutorial.yammer_events e
      on e.user_id = ex.user_id
      and e.occurred_at >= '2014-05-01'
      and e.occurred_at < '2014-06-01'
      and e.event_name = 'send_message'
      where ex.experiment_group = 'test_group'
      group by 1) x
```

<img width="417" alt="image" src="https://user-images.githubusercontent.com/32259752/178141672-4bb8e08a-627d-4577-a482-a199ad491312.png">

```
SELECT 
count(user_id),
avg(count_engagement) as avg_message,
stddev_samp(count_engagement) as std_message
FROM (SELECT 
        distinct ex.user_id,
        COUNT(e.event_name) AS count_engagement
      from tutorial.yammer_experiments ex 
      left join tutorial.yammer_events e
      on e.user_id = ex.user_id
      and e.occurred_at >= '2014-05-01'
      and e.occurred_at < '2014-06-01'
      and e.event_name = 'send_message'
      where ex.experiment_group = 'control_group'
      group by 1) x
```

<img width="413" alt="image" src="https://user-images.githubusercontent.com/32259752/178141700-3eb18e1e-678b-4ec1-9a22-7fddb5c32081.png">

I query the average number of messages of users in treatment and test group posted before the experiment (2014-05-01 to 2014-05-31) to see 
if users in different groups had comparable level of engagement.

Nul hypothesis: There was no difference in the average number of messages between control and treatment group.  
Alternative hypothesis: The average number of messages was greater for treatment group than for control group, which means users in treatment already 
were more engaged in posting messages. This is a one-tailed hypothesis, and I will use t-score.

After calculation, the P-Value is .000117, and the result is significant at alpha = 0.05. It means the treatment group is different from control group in 
average number of messages. It is possible that users in two groups were not randomly assigned. Their 
previous user attribute (in this case, engagement in posting message) could have an impact on the result of the experiment. This could be a problem 
with the underlying experimental design, infrastructure, or data processing.

## External validity
Even if the result of the experiment is valid, that does not mean we can generalize the result to all users over time. We cannot assume that an ideal 
result at this time for this segment of users will apply to all time and all users. 

### Is this rise in the average number of messages per user a novelty effect? Users used it much more than before simply because it was a new feature, rather than because they needed it.
**Novelty effect** means when a new feature is introduced, initially it attracts users to try it. If users do not find the feature userful, repeat 
usage will be small. A treatment may appear to perfrom well at first, but it is possible the treatment will quickly decline over time. So, I want to know
over the time of this one-month experiment, did users gradully lose their interest in the new feature?

```
with treatment as (
select 
  case when e.occurred_at >= '2014-06-01' AND e.occurred_at < '2014-06-08' THEN 'week_1'
       when e.occurred_at >= '2014-06-08' AND e.occurred_at < '2014-06-15' THEN 'week_2'
       when e.occurred_at >= '2014-06-15' AND e.occurred_at < '2014-06-22' THEN 'week_3'
       when e.occurred_at >= '2014-06-22' AND e.occurred_at < '2014-07-01' THEN 'week_4' end as week,
  count(e.event_name)*1.00/count(distinct ex.user_id) as treatment_messages_per_user
from tutorial.yammer_experiments ex 
left join tutorial.yammer_events e
on e.user_id  = ex.user_id 
and e.event_name = 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'test_group'
group by 1
)

, control as (
select 
  case when e.occurred_at >= '2014-06-01' AND e.occurred_at < '2014-06-08' THEN 'week_1'
       when e.occurred_at >= '2014-06-08' AND e.occurred_at < '2014-06-15' THEN 'week_2'
       when e.occurred_at >= '2014-06-15' AND e.occurred_at < '2014-06-22' THEN 'week_3'
       when e.occurred_at >= '2014-06-22' AND e.occurred_at < '2014-07-01' THEN 'week_4' end as week,
  count(e.event_name)*1.00/count(distinct ex.user_id) as control_messages_per_user
from tutorial.yammer_experiments ex 
left join tutorial.yammer_events e 
on e.user_id  = ex.user_id 
and e.event_name = 'send_message'
and e.occurred_at >= '2014-06-01'
and e.occurred_at < '2014-07-01'
where ex.experiment_group = 'control_group'
group by 1
)

select t.week, t.treatment_messages_per_user, c.control_messages_per_user
from treatment t 
join control c 
on t.week = c.week
```

<img width="592" alt="image" src="https://user-images.githubusercontent.com/32259752/178147910-1b2d32cd-5cf2-4850-a64b-3b0fd41aef86.png">

In this query, I divided June into four weeeks. From the result, I can see that users' engagement in treatment group were constantly greater 
than those in control group. So the result of the experiment was not a case of novelty effect.

```
with treatment as (
select 
  date_trunc('week', e.occurred_at) as week,
  count(e.event_name),
  count(e.event_name)*1.00/count(distinct e.user_id) as treatment_messages_per_user
from tutorial.yammer_events e
join tutorial.yammer_experiments ex 
on e.user_id  = ex.user_id 
and e.event_name = 'send_message'
and e.occurred_at >= '2014-07-01'
and e.occurred_at < '2014-08-01'
and ex.experiment_group = 'test_group'
group by 1
)

, control as (
select 
  date_trunc('week', e.occurred_at) as week,
  count(e.event_name),
  count(e.event_name)*1.00/count(distinct e.user_id) control_messages_per_user
from tutorial.yammer_events e
join tutorial.yammer_experiments ex 
on e.user_id  = ex.user_id 
and e.event_name = 'send_message'
and e.occurred_at >= '2014-07-01'
and e.occurred_at < '2014-08-01'
and ex.experiment_group = 'control_group'
group by 1
)

select 
  treatment.week,
  treatment.treatment_messages_per_user,
  control.control_messages_per_user
from treatment 
join control 
on treatment.week = control.week
```

<img width="583" alt="image" src="https://user-images.githubusercontent.com/32259752/178148106-538d0a34-8e84-4676-9f01-0976680218e0.png">

Even after the experiment in July, I still observe that users in treatment were posting more messages than those in control group.

### Can we observe the rise in all countries?
Analyzing the success of the new feature by different segments can provide interesting insights and lead to discoveries. In here, I want to know if all 
countries were having the same level of surge in the average number of messages per user.

```
with treatment as (select 
  ex.location as location,
  count(e.event_name)*1.00/count(distinct ex.user_id) as treatment_average_number_of_messages
from tutorial.yammer_experiments ex
join tutorial.yammer_events e 
on ex.user_id = e.user_id
and e.event_name = 'send_message'
where ex.experiment_group = 'test_group'
group by 1
order by 2 desc)

, control as (select 
  ex.location as location,
  count(e.event_name)*1.00/count(distinct ex.user_id) as control_average_number_of_messages
from tutorial.yammer_experiments ex
join tutorial.yammer_events e 
on ex.user_id = e.user_id
and e.event_name = 'send_message'
where ex.experiment_group = 'control_group'
group by 1
order by 2 desc)

select treatment.location, treatment.treatment_average_number_of_messages, control.control_average_number_of_messages,
(treatment.treatment_average_number_of_messages - control.control_average_number_of_messages) as difference
from treatment 
join control 
on treatment.location = control.location
order by 4 desc
```

<img width="810" alt="image" src="https://user-images.githubusercontent.com/32259752/178148499-541432d6-2065-4a27-9bd3-124eca4ebb3c.png">

We can see that users in **Iraq** and **Poland** had had an unusual difference between treatment and control group. 
Then I went on investigating if this was something usual.

```
with treatment as (select 
 ex.location, date_trunc('month', e.occurred_at) as month,
 count(e.event_name)*1.00/count(distinct ex.user_id) as treatment_average_number_of_messages
from tutorial.yammer_experiments ex 
left join tutorial.yammer_events e
on e.user_id = ex.user_id 
and e.event_name = 'send_message'
where ex.experiment_group = 'test_group'
and ex.location in ('Iraq', 'Poland')
group by 1,2)

, control as (select 
 ex.location, date_trunc('month', e.occurred_at) as month,
 count(e.event_name)*1.00/count(distinct ex.user_id) as control_average_number_of_messages
from tutorial.yammer_experiments ex 
left join tutorial.yammer_events e
on e.user_id = ex.user_id 
and ex.location in ('Iraq', 'Poland')
and e.event_name = 'send_message'
where ex.experiment_group = 'control_group'
and ex.location in ('Iraq', 'Poland')
group by 1,2)

select treatment.location, treatment.month, treatment.treatment_average_number_of_messages, control.control_average_number_of_messages, 
treatment.treatment_average_number_of_messages-control.control_average_number_of_messages as difference
from treatment 
left join control 
on treatment.month = control.month
and treatment.location = control.location 
order by 1,2
```

<img width="904" alt="image" src="https://user-images.githubusercontent.com/32259752/178148742-c05ac998-85fb-454f-83e6-7b26467180c1.png">

We can see that behavior of users in Iraq was suspicious. For those in treatment group, they posted about 15 to 16 messages per use in May and June, but 
the number plummeted down to 2.5 to 3 in July and August. For those in control group, they barely posted any messages. There could be some technical 
problems in logging system in Iraq, seasonality in the region, or even because of robots. Until further investigation into this anamoly, 
I would not say that the result of the experiment could be applies to Iraq.

Compared with Iraq, user behavior in Poland made more sense. Users in Poland have used the message feature even before the introduction of the new feature, 
and they kept using this function even after the experiment. 

### Can we observe the rise in all devices?
Different operating systems and devices used different tracking methodologies. While the tracking system for messages works for some systems, 
it may fail in some other platforms. So, I want to if there is any suspicious device that could threaten the validity of the result.

```
with treatment as (select 
  ex.device as device,
  count(e.event_name)*1.00/count(distinct ex.user_id) as treatment_average_number_of_messages
from tutorial.yammer_experiments ex
left join tutorial.yammer_events e 
on ex.user_id = e.user_id
where ex.experiment_group = 'test_group'
and e.event_name = 'send_message'
and e.occurred_at >= '2014-06-01' 
and e.occurred_at < '2014-07-01'
group by 1
order by 2 desc)

, control as (select 
  ex.device as device,
  count(e.event_name)*1.00/count(distinct ex.user_id) as control_average_number_of_messages
from tutorial.yammer_experiments ex
left join tutorial.yammer_events e 
on ex.user_id = e.user_id
where ex.experiment_group = 'control_group'
and e.event_name = 'send_message'
and e.occurred_at >= '2014-06-01' 
and e.occurred_at < '2014-07-01'
group by 1
order by 2 desc)

select treatment.device, treatment.treatment_average_number_of_messages, control.control_average_number_of_messages,
(treatment.treatment_average_number_of_messages - control.control_average_number_of_messages) as difference,
avg(treatment.treatment_average_number_of_messages - control.control_average_number_of_messages) OVER () AS avg_difference
from treatment 
join control 
on treatment.device = control.device
order by 4 desc
```

<img width="947" alt="image" src="https://user-images.githubusercontent.com/32259752/178149094-3c481b81-34b5-4907-9a9f-167f50e0360e.png">

You can see that while the average difference between treatment and control group is 1.1942, **Mac Mini** and **ASUS Chromebook** had a 
greater difference, which is unusual compared to other platforms. Before further investigation, I would not apply the result of the experiment 
to users of these two platforms.

### Can we observe the rise in users with different activation dates?
When new users come to Yammer, they could be overwhelmed by all the services provided, or they do not have many connections. I want to know the activation 
date of users in Yammer. 

```
select 
 date_trunc('month', u.activated_at) as activate_month,
 count(case when ex.experiment_group = 'test_group' then 1 else null end) as count_treatment,
 count(case when ex.experiment_group = 'control_group' then 1 else null end) as count_control
from tutorial.yammer_experiments ex
left join tutorial.yammer_users u 
on u.user_id = ex.user_id 
group by 1
order by 1
```

<img width="434" alt="image" src="https://user-images.githubusercontent.com/32259752/178289498-6594805c-75ca-4f00-8b65-e012b765b615.png">

I do not observe significant difference between treatment and control group from 2013-01 to 2014-05, but in 2014-06, all the new users were assigned to 
control group. It is possible that these new users were not familiar with Yammer services and had fewer connections in Yammer than older users in 
treatment group. This finding confirmed my suspicion that users were not randomly assigned to treatment and control group.

# Summary
The result of the experiment is valid, we can say with confidence that the new version of Publisher is more effective in engaging users than 
the old version. But the key problem of the experiment is the randomization unit. In this experiment, users should be the randomization unit. 
However, this is not the case here. New users were more likely to be assigned to control group, making treatment and control group statistically dissimilar.
Before further investigation, I will not say the increase in the number of messages users published was only because of the new version of Publisher. 
Also, it is not safe to apply the result to users in Iraq, or users using Mac Mini** and ASUS Chromebook. 
These segment are not representative of all users.





