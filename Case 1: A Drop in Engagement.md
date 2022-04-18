# 1. What does it mean by "a drop in engament"?
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

Combining these two queries, we can see that we are not only losing customers, but also customers' engagements. Customers do not use Yammer services as before.

```
SELECT EXTRACT(MONTH from occurred_at) AS month, COUNT(DISTINCT user_id) AS count_of_signup_user
FROM tutorial.yammer_events 
WHERE event_type = 'signup_flow'
GROUP BY 1
```

<img width="344" alt="image" src="https://user-images.githubusercontent.com/32259752/161052446-c15d0790-6390-4d5f-acaf-d9cd58b6dd5c.png">

But the signup flow in August is higher than ever before. This is at odds with the drop in engagement. So, it is possible that this engagement drop is because of 
technical issues, like power outage, 
a bug in our engagement logging system, etc. 

***

# 2. Slice the engagement into smaller segment

## Language
Ymmmer provides service in **12** languages: Portuguese, Indian, Russian, Japanese, German, French, Arabic, Chinese, Korean, Italian, English, Spanish. 
Here, I want to see which language did we lose the most customers.

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
So, I would start out investigating if our services provided by these three languages are up to the expectations of our customers.

## Company
```SELECT COUNT(DISTINCT company_id)
FROM tutorial.yammer_users
```

There are **13198** companies are using or used Yammer services. There are **2808** companies still use Yammer service in August, but only **385**
companies used Yammer throughout all 4 months. I want to see which companies contributed most to the loss of our customers in August.

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


Company 2, 6, 7, 12, 1 are the key customer. Not only becuase they are big customers, we also lost a great number of users from them. I would investigate the reason why these companies are starting to lose their interests in using Yammer services.

## Type of Event
There are 21 differnt kinds of events for Yammer clients to be engaged in. Since we are losing engaging clients, I want to figure out in which engagement event did we lose the most clients.

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

## Location
Geological location could be a contributing factor to our loss of engaging clients because of local holidays. We have clients from 47 countries over the world.
It is possible that we lost a lot of clients in one location in August because of seaonality. For example, Northern hemisphere countries could be having 
summer vacation so that they use Yammer services, a business social network, less often.

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

At first glance, we can see that we lost most clients in the United States, France, and Canada. So, I will start off to investigate into is there anything could be responsibile for our drop in customers in 
these three countries, whether it is about their national holidays, our services, or if there was a competitor appearing in these countries providing 
similar service.

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

We lost most clients in the United States using the language English. Then it is French clients using the language French, 
and Italian clients using the language Italian. English language users in United States are especially worth notice taking. We lost
much more clients in this segment than others.

## Device
Now, we have a variety of devices to carry out business: computer, tablet, phones. Or even different operating system: Mac OS, Windows, Android, 
iOS etc. Different versiosn of Yammer service applies to different devices and operating systems. There are totally 26 different devices: 
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

We have lost most clients using iPhone 5, and then it is Samsung Galaxy S4, iPhone 5s. Possibly it is because something wrong with the phone app, 
or our service is no longer avaiable on App Store.

I want to dive deeper into different segments of different devices. First, I divided these 26 different devices into three differnt platforms: computer, phone, and tablet.
It could be that our services changed in some way that made some platfroms could not perform as good as before so that clients were driven away.

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

Most lost clients are from tablet segment. We could investigate into this segment to see that if our tablet services are not satisfactory to our clients.






