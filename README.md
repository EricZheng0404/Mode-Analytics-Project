# Mode-Analytics-Training
## Analytics Case: Yammer
Yammer is a social network for communicating with coworkers. Individuals share documents, updates, and ideas by posting them in groups. Yammer is free to use indefinitely, but companies must pay license fees if they want access to administrative controls, including integration with user management systems like ActiveDirectory.

Yammer has a centralized Analytics team, which sits in the Engineering organization. Their primary goal is to drive better product and business decisions using data. They do this partially by providing tools and education that make other teams within Yammer more effective at using data to make better decisions. They also perform ad-hoc analysis to support specific decisions.

## Cases
**1. A Drop in Engagement**\
Engagement dips—you figure out the source of the problem.

**2. Understanding Search**\
The product team is thinking about revamping search. Your job is to figure out whether they should change it at all, and if so, what should be changed.

**3. The Best A/B Test Ever**
A new feature tests off the charts. Your job is to determine the validity of the experiment.

## Available Datasets
**Table 1: Users**\
This table includes one row per user, with descriptive information about that user's account.
| Column | Description |
| --- | --- |
|user_id    | An unique ID per user. Can be joined to user_id in either of the other tables|
|created_at | The time the user was created (first signed up)|
|state|	The state of the user (active or pending)|
|activated_at|	The time the user was activated, if they are active|
|company_id|	The ID of the user's company|
|language|	The chosen language of the user|

**Table 2: Events**\
This table includes one row per event, where an event is an action that a user has taken on Yammer. These events include login events, messaging events, search events, events logged as users progress through a signup funnel, events around received emails.
| Column | Description |
| --- | --- |
|user_id|	The ID of the user logging the event. Can be joined to user\_id in either of the other tables
|occurred_at|	The time the event occurred|
|event_type|	The general event type. There are two values in this dataset: "signup_flow", which refers to anything occuring during the process of a user's |authentication, and "engagement", which refers to general product usage after the user has signed up for the first time|
|event_name|	The specific action the user took. Possible values include: **create_user**: User is added to Yammer's database during signup process **enter_email**: User begins the signup process by entering her email address **enter_info**: User enters her name and personal information during signup process **complete_signup**: User completes the entire signup/authentication process **home_page**: User loads the home page **like_message**: User likes another user's message **login**: User logs into Yammer **search_autocomplete**: User selects a search result from the autocomplete **list search_run**: User runs a search query and is taken to the search results **page search_click_result_X**: User clicks search result X on the results page, where X is a number from 1 through 10. **send_message**: User posts a message view_inbox: User views messages in her inbox|
|location|	The country from which the event was logged (collected through IP address)|
|device|	The type of device used to log the event|

**Table 3: Email Events**\
This table contains events specific to the sending of emails. It is similar in structure to the events table above.
| Column | Description |
| --- | --- |
|user_id|	The ID of the user to whom the event relates. Can be joined to user_id in either of the other tables|
|occurred_at|	The time the event occurred|
|action|	The name of the event that occurred. "sent_weekly_digest" means that the user was delivered a digest email showing relevant conversations from the previous day. "email_open" means that the user opened the email. "email_clickthrough" means that the user clicked a link in the email. "sent_reengagement_email" means that the inactive user received an re-engagement email.|
