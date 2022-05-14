# 1. Problem
Yammer was founded 2008 as a freemium enterprise social networking service used for private communication within organizations. It was acquired by Microsoft for $1.2 billion in 2012 and is now available in all Office 365 products.

This is a case study of Yammer’s drop in user engagement from May to August of 2014. Due to privacy reasons, the data used is not real and did not reflected real user engagement of the Yammer app at the time. You can [refer here for the case study and data.](https://mode.com/sql-tutorial/a-drop-in-user-engagement/)

I am responsible for determining what caused the dip at the end of July as shown below, and recommending solutions for the problem.

# 2. Context
At the later half of August, Yammer detected a drop in overall user engagement through a graph that looks like this:
<img width="773" alt="image" src="https://user-images.githubusercontent.com/32259752/164892249-19603da3-223a-426f-adda-d764578298cd.png">

The above chart shows the number of engaged users each week. Yammer defines engagement as having made some type of server call by interacting with the product (shown in the data as events of type "engagement"). Any point in this chart can be interpreted as "the number of users who logged at least one engagement event during the week starting on that date." As it is shown in the graph, the number of active users were steadily increasing from April to the end of July. But it started to sharply fall by almost 200 users, and remained at a lower level throughtout August.

So, what does it mean by "a drop in engament"?
```
SELECT EXTRACT(MONTH from occurred_at) AS month, COUNT(user_id) AS count_of_engagement
FROM tutorial.yammer_events 
WHERE event_type = 'engagement'
GROUP BY 1
```

<img width="294" alt="image" src="https://user-images.githubusercontent.com/32259752/161051290-bc14adf1-8632-478b-b424-310c782eadc3.png">

```
SELECT EXTRACT(MONTH from occurred_at) AS month, COUNT(DISTINCT user_id) AS count_of_engaging_customer
FROM tutorial.yammer_events 
WHERE event_type = 'engagement'
GROUP BY 1
```

<img width="294" alt="image" src="https://user-images.githubusercontent.com/32259752/161051391-5225f297-7049-4fdc-84f9-3826abc81a49.png">

Combining these two queries, we can see that we are not only losing some individual customers, but also individual customers' engagements. Customers do not use Yammer services frequently as before.


# 3. Identify potential causes
* **System bug**: There might be some system bugs interfering with users that discourage them from our services. Some system bugs includes power outage, bugs in logging code, someone checked in a code without testing, forgot to pay a license fee, and the thrid-party software stopped working etc.
* **Local competitors**: It’s possible that a competitor launched a competing product at the end of July that gain the traction of our clients.
* **Language**: Ymmmer provides service in **12** languages: Portuguese, Indian, Russian, Japanese, German, French, Arabic, Chinese, Korean, Italian, English, Spanish. It is possible that our services in some languages are not satisfactory to our clients.
* **Company**: There are **13198** companies are using or used Yammer services. There are **2808** companies still use Yammer service in August, but only **385** companies used Yammer throughout all 4 months. It could be that our services do not cater to some companies' needs any more.
* **Type of event**: There are 21 differnt kinds of events for Yammer clients to be engaged in. Since we are losing engaging clients, I want to figure out in which engagement event did we lose the most clients.
* **Location**: Geological location could be a contributing factor to our loss of engaging clients because of local holidays. We have clients from 47 countries over the world.
It is possible that we lost a lot of clients in one location in August because of seaonality. For example, Northern hemisphere countries could be having 
summer vacation so that their usage of Yammer services, a business social network, is not as much.
* **Device**: Yammer customers can access to Yammer services through different platforms: computer, tablet, and phones. Or even different operating system: Mac OS, Windows, Android, iOS etc. Different versiosn of Yammer service applies to different devices and operating systems. Yammer app on some devices may be glitchy. We can see if user engagement was affected by one or multiple types of devices.
* **Email service**: Yammer regularly sends emails to clients. I want to see if this email service contributed to the engagement drop. It is possible that our clients are turning away from Yammer because they received too many (low-quality) emails. I also want to see if email open rate and clickthrough rate have anything to do with the client loss.

# 4. Investigating the date

## System bug
```
SELECT EXTRACT(MONTH from occurred_at) AS month, COUNT(DISTINCT user_id) AS count_of_signup_user
FROM tutorial.yammer_events 
WHERE event_type = 'signup_flow'
GROUP BY 1
```

<img width="344" alt="image" src="https://user-images.githubusercontent.com/32259752/161052446-c15d0790-6390-4d5f-acaf-d9cd58b6dd5c.png">

The signup flow in August is higher than ever before. This is at odds with the drop in engagement. So, it is possible that the engagement drop is because of technical issues, like power outage, a bug in our engagement logging system, etc.

## Local competitor
From the previous query, we can also see that there should be no local competitor to drive our clients away because there were more people signing up for our service.

## Language
```
WITH sub AS (SELECT EXTRACT(MONTH from occurred_at) AS month, u.language, COUNT(DISTINCT e.user_id) AS user_number
FROM tutorial.yammer_events e
JOIN tutorial.yammer_users u
ON e.user_id = u.user_id
WHERE e.event_type = 'engagement'
GROUP BY 1, 2
ORDER BY 2,1
)

SELECT month, language, user_number, LAG(user_number) OVER (PARTITION BY language ORDER BY month) AS pre_month_count, 
(user_number - LAG(user_number) OVER (PARTITION BY language ORDER BY month) ) AS user_loss
FROM sub
ORDER BY 5
```

<img width="623" alt="image" src="https://user-images.githubusercontent.com/32259752/161056461-ea603711-283f-45c0-92c4-81389f2a3b43.png">

The lost customers are mostly represnted by customers using **English, French, and Italian**. 
So, I would start out investigating if our services provided in these three languages are up to the expectations of our customers.

## Company
```
WITH sub AS (SELECT EXTRACT(MONTH from occurred_at) AS month, u.company_id, COUNT(DISTINCT e.user_id) AS user_number
FROM tutorial.yammer_events e
JOIN tutorial.yammer_users u
ON e.user_id = u.user_id
WHERE e.event_type = 'engagement'
GROUP BY 2, 1
)


, sub2  AS (SELECT month, company_id, user_number, LAG(user_number) OVER (PARTITION BY company_id ORDER BY month) AS previous_month_customer_count,
(user_number - LAG(user_number) OVER (PARTITION BY company_id ORDER BY month)) AS customer_loss
FROM sub
ORDER BY 5 )

SELECT *
FROM sub2
WHERE month = 8
```

<img width="759" alt="image" src="https://user-images.githubusercontent.com/32259752/161050895-de91b89e-d686-403f-b654-cb925ecae59a.png">

**Company 2, 6, 7, 12, 1 are the key customer.** Not only becuase they are big customers, we also lost a great number of users from them. I would investigate the reason why these companies are starting to lose their interests in using Yammer services.

## Type of Event
```
WITH sub AS (SELECT EXTRACT(MONTH from occurred_at) AS month, event_name, COUNT(DISTINCT user_id) AS user_number
FROM tutorial.yammer_events 
WHERE event_type = 'engagement'
GROUP BY 2,1
)

, sub2 AS (SELECT month,event_name, user_number, LAG(user_number) OVER (PARTITION BY event_name ORDER BY month) AS previous_month_customer_count,
(user_number - LAG(user_number) OVER (PARTITION BY event_name ORDER BY month)) AS customer_loss
FROM sub
ORDER BY 5 )

SELECT *
FROM sub2
WHERE month = 8
```

<img width="782" alt="image" src="https://user-images.githubusercontent.com/32259752/161048281-14c60915-c8e7-4e50-98e9-3df4e41f7028.png">

We lost most clients in **home_page, login, view_box, send_message, like_message**. 
So, we need to inquire into if these services are problematic that they turn the clients away. 
Also, clients are not uisng out serarch funtion that much compared
to other engagements. We can try to improve our search funtion to boost engagement rate.

Going a step further, I want to understand our big customers' need better and to know which type of event our big customers lost interest in so that we can improve that specific areas for them.

```
WITH sub AS 
(
SELECT 
  EXTRACT('month' from e.occurred_at) AS month,
  u.company_id,
  e.event_name,
  COUNT(*) AS number_of_engagement
FROM tutorial.yammer_users u  
JOIN tutorial.yammer_events e 
ON u.user_id = e.user_id 
WHERE e.event_type = 'engagement'
GROUP BY 2,3,1
ORDER BY 2,3,1
)

,sub2 AS 
(SELECT
  *,
  LAG(number_of_engagement) OVER (PARTITION BY company_id, event_name ORDER BY month) AS previous_month_customer_count,
  (number_of_engagement - LAG(number_of_engagement) OVER (PARTITION BY company_id, event_name ORDER BY month)) AS customer_loss
FROM sub
ORDER BY company_id, event_name, month, customer_loss)

SELECT *
FROM sub2
WHERE month = 8
ORDER BY customer_loss
```

<img width="962" alt="image" src="https://user-images.githubusercontent.com/32259752/168414871-057a27ed-4f49-42d1-b3e9-e16da302bc8e.png">

We can see that **home_page, login, view_box, send_message, like_message** are also the types of event that our big customers use much less than before.

## Location
```
WITH sub AS (SELECT EXTRACT(MONTH FROM occurred_at) AS month, e.location,  COUNT(DISTINCT e.user_id) AS count_of_user
FROM tutorial.yammer_events e
JOIN tutorial.yammer_users u
ON e.user_id = u.user_id
GROUP BY 2,1)


, sub2 AS (SELECT *, LAG(count_of_user) OVER (PARTITION BY location ORDER BY month) AS pre_month_count_of_user, 
      (count_of_user - LAG(count_of_user) OVER (PARTITION BY location ORDER BY month) ) AS customer_loss
FROM sub)

SELECT *
FROM sub2
WHERE month = 8
ORDER BY 5
```

At first glance, we can see that we lost most clients in the United States, France, and Canada. So, I will start off to investigate into if there is anything that is responsibile for our drop in customers in these three countries. It would be of use to check with other teams if there were any new releases or product upgrades localized to United States and Canada.

Sometimes in a single location, many languages are used. For example, In the United States, four languages are used: Chinese, French, Spanish, and English. 
I want to know in a specific country, in which language did we lose the most clients that I could understand our 

```
WITH sub AS (SELECT EXTRACT(MONTH FROM occurred_at) AS month, location, u.language, COUNT(DISTINCT e.user_id) AS count_of_user
FROM tutorial.yammer_events e
JOIN tutorial.yammer_users u
ON e.user_id = u.user_id
GROUP BY 1,2,3)


, sub2 AS (SELECT *, LAG(count_of_user) OVER (PARTITION BY location, language ORDER BY month)  AS pre_month_count_of_user,
(count_of_user - LAG(count_of_user) OVER (PARTITION BY location, language ORDER BY month)) AS customer_loss
      
FROM sub)

SELECT *
FROM sub2
WHERE month = 8
ORDER BY 6 
```

<img width="844" alt="image" src="https://user-images.githubusercontent.com/32259752/161246901-204adddc-6811-4b9a-9049-fc74509cb77a.png">

**We lost most clients in the United States using the language English.** Then it is French clients using the language French, 
and Italian clients using the language Italian. English language users in United States are especially worth notice taking. We lost
much more clients in this segment than others.

## Device
There are totally 26 different devices: 
Acer Aspire Desktop, Acer Aspire Notebook, Amazon Fire Phone, Asus Chromebook, Dell Inspiron Desktop, Dell Inspiron Notebook, Hp Pavilion Desktop, Htc One, Ipad Air, 
iPad Mini, iPhone 4s, iPhone 5, iPhone 5s, Kindle Fire, Lenovo Thinkpad, Macbook Air, Macbook Pro, Mac Mini, Nexus 10, Nexus 5, Nexus 7, Nokia Lumia 635, Samsumg Galaxy Tablet, 
Samsung Galaxy Note, Samsung Galaxy S4. 

```
SELECT DISTINCT device 
FROM tutorial.yammer_events
WHERE EXTRACT(MONTH from occurred_at) = 8
```

<img width="180" alt="image" src="https://user-images.githubusercontent.com/32259752/161257655-880e9b60-87e2-4bb9-9948-10fcb49b6663.png">


First, I want to see which device lost the most clients.

```
WITH sub AS (SELECT EXTRACT(MONTH from occurred_at) AS month, device, COUNT(DISTINCT user_id) AS user_number
FROM tutorial.yammer_events 
WHERE event_type = 'engagement'
GROUP BY 2,1
)

, sub2 AS (SELECT *, LAG(user_number) OVER (PARTITION BY device ORDER BY month) AS previous_month_customer_count,
(user_number - LAG(user_number) OVER (PARTITION BY device ORDER BY month)) AS customer_loss
FROM sub
ORDER BY 5 )

SELECT *
FROM sub2
WHERE month = 8
```

<img width="790" alt="image" src="https://user-images.githubusercontent.com/32259752/161255941-a201b163-fd3a-4b71-b20e-e8654a1c078f.png">

**We have lost most clients using iPhone 5, and then it is Samsung Galaxy S4, iPhone 5s.** Possibly it is because something is wrong with the phone app, 
or our service is no longer avaiable on App Store, or these devices are no longer available in the market.

I want to dive deeper into different segments of different devices. First, I divided these 26 different devices into three differnt categories: computer, phone, and tablet.
It could be that our services changed in some way that made some categories could not perform as good as before so that clients were driven away.

* Computer: \
acer aspire desktop, acer aspire notebook, asus chromebook, dell inspiron desktop, dell inspiron notebook, hp pavilion desktop, lenovo thinkpad, macbook air, macbook pro, mac mini
* Phone: \
amazon fire phone, htc one, iphone 4s, iphone 5, iphone 5s,  nokia lumia 635, samsung galaxy note, samsung galaxy s4
* Tablet:\
ipad air, ipad mini, kindle fire, nexus 10, nexus 5, nexus 7, samsumg galaxy tablet

```
WITH sub AS (
  SELECT EXTRACT(MONTH from occurred_at) AS month, 
        CASE WHEN device in ('acer aspire desktop', 'acer aspire notebook', 'asus chromebook', 'dell inspiron desktop', 'dell inspiron notebook', 'hp pavilion desktop', 'lenovo thinkpad', 'macbook air', 'macbook pro', 'mac mini', 'windows surface') THEN 'computer' 
        WHEN device in ('amazon fire phone', 'htc one', 'iphone 4s', 'iphone 5', 'iphone 5s', 'nokia lumia 635', 'samsung galaxy note', 'samsung galaxy s4') THEN 'phone'
        WHEN device in ('ipad air', 'ipad mini', 'kindle fire', 'nexus 10', 'nexus 5', 'nexus 7', 'samsumg galaxy tablet') THEN 'tablet' END AS category
         , COUNT(DISTINCT user_id) AS user_number
  FROM tutorial.yammer_events 
  WHERE event_type = 'engagement'
  GROUP BY 2,1
)

, sub2 AS (SELECT *, LAG(user_number) OVER (PARTITION BY category ORDER BY month) AS previous_month_customer_count,
(user_number - LAG(user_number) OVER (PARTITION BY category ORDER BY month)) AS customer_loss
FROM sub
ORDER BY 5 )

SELECT *
FROM sub2
WHERE month = 8
```

<img width="703" alt="image" src="https://user-images.githubusercontent.com/32259752/161410525-83d2cb7f-b789-4a9e-8da6-6e09c057ddf1.png">

**Most lost clients are mostly represented by phone users**, potentially due to UX design problems or dysfunctional features leadng to discontent. We could investigate into this category to see that if our phone services are satisfactory to our clients.

## Email service
```
SELECT EXTRACT('month' FROM occurred_at) AS month,
       SUM(CASE WHEN action = 'email_open' THEN 1 ELSE 0 END) AS email_open,
       SUM(CASE WHEN action = 'email_clickthrough' THEN 1 ELSE 0 END) AS email_clickthrough,
       SUM(CASE WHEN action = 'sent_weekly_digest' THEN 1 ELSE 0 END) AS sent_weekly_digest,
       SUM(CASE WHEN action = 'sent_reengagement_email' THEN 1 ELSE 0 END) AS sent_reengagement_email
FROM tutorial.yammer_emails
GROUP BY 1
ORDER BY 1
```

<img width="784" alt="image" src="https://user-images.githubusercontent.com/32259752/165688693-b12b5485-e8e2-480c-bd33-d27897d114b4.png">

```
WITH sub AS (SELECT EXTRACT('month' FROM occurred_at) AS month,
       COUNT(DISTINCT user_id) AS count_user,
       SUM(CASE WHEN action = 'email_open' THEN 1 ELSE 0 END) AS email_open,
       SUM(CASE WHEN action = 'email_clickthrough' THEN 1 ELSE 0 END) AS email_clickthrough,
       SUM(CASE WHEN action = 'sent_weekly_digest' THEN 1 ELSE 0 END) AS sent_weekly_digest,
       SUM(CASE WHEN action = 'sent_reengagement_email' THEN 1 ELSE 0 END) AS sent_reengagement_email
FROM tutorial.yammer_emails
GROUP BY 1
ORDER BY 1)

SELECT month,
       count_user,
       (email_open*1.0/count_user) AS email_open_per_user,
       (email_clickthrough*1.0/count_user) AS email_clickthrough_per_user,
       (sent_weekly_digest*1.0/count_user) AS sent_weekly_digest_per_user,
       (sent_reengagement_email*1.0/count_user) AS sent_reengagement_email_per_user
FROm sub
```

<img width="1101" alt="image" src="https://user-images.githubusercontent.com/32259752/165692396-299ee8f7-e20a-43e2-9829-58299a2d16cd.png">

From these two queries, we can see that although there was an increase in the numbers of email open, sent weekly digest, and sent reengagement email, the average per user of these actions actually decreased due to the incoming clients in August. **This signals that we sent less emails to per clients in August than ever before, and fewer users are interested in opening or clicking on the emails.** This could be a possible cause for the drop of engagement in August.

Going a step further, I want to see if we can narrow the problem down by figuring out which type of email is less welcomed by the clients.

```
WITH sub AS (
SELECT occurred_at,
EXTRACT('month' FROM occurred_at) as month,
       user_id,
       action,
       LEAD(action, 1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS opened_email,
       LEAD(action, 2) OVER (PARTITION BY user_id ORDER BY occurred_at) AS clicked_email
FROM tutorial.yammer_emails
)

SELECT 
  action,
  month,
  count(action),
  SUM(CASE WHEN opened_email = 'email_open' THEN 1 ELSE 0 END) AS opened_email,
  SUM(CASE WHEN clicked_email = 'email_clickthrough' THEN 1 ELSE 0 END) AS clicked_email
FROM sub
WHERE action in ('sent_weekly_digest','sent_reengagement_email')
GROUP BY
  action,
  month
ORDER BY
  action,
  month
```

<img width="723" alt="image" src="https://user-images.githubusercontent.com/32259752/167878320-ee11c4cf-3662-4049-a0d8-726a0ce5d3f7.png">

We can see that Yammer was sending out more re-engagement emails and weekly digests more over the past several month and an increasing number of users opened these emails, **but users are less interested in clicking open the weely digest than even before.** This means that the weekly digest content is not relevant enough to our users, or the intended user action is not explicit enough. This could make the users disappoint at our services and thus drive the engagement drop.

# 5. Recommendations
1. Reach out to technical team to see if the drop in engagement is because of **system bugs**
2. Investigate our service in **English, French, and Italian**, especially for those users who use these languages in **United States**
3. Check if our big customers if satisfactory with our services: **Company 2, 6, 7, 12, 1**
4. See if we can improve the engagements of **home_page, login, view_box, send_message, like_message**
5. Look into our services provided on the phone, especially those clients who use **iPhone 5, Samsung Galaxy S4, and iPhone 5s**
6. **Improve our mail quality**, especially the weekly digest, to get more users to open and click through the email

# 6. Further analysis
1. Survey for user feedback on weekly digest
2. A/B testing for weekly digest email content
3. Cohort analysis to see if the cause if due to a short user lifecycle
