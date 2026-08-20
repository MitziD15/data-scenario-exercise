1. First response time by team Write a query returning the average first_response_minutes for tickets closed in the last 30 days, grouped by agent team.

```sql
SELECT agents.team, AVG(tickets.first_response_minutes)
FROM tickets
JOIN agents ON tickets.agent_id = agents.agent_id
WHERE tickets.status= "closed"
AND tickets.closed_at>= CURRENT_DATE - INTERVAL "30 days"
GROUP by agents.team;
```


2. Agents with above-average reopen rates Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.
```sql
SELECT agent_id, team, reopened_tickets^1.0/total_tickets AS reopen_rate
INTO agent_reopen_rates
FROM(
      SELECT tickets.agent_id, agents.team,
          COUNT(*) AS total_tickets,
          SUM(CASE WHEN tickets.reopened_count>0 THEN 1 ELSE 0 END) AS reopened_tickets
      FROM tickets
      JOIN agents ON tickets.agent_id = agents.agent_id
      GROUP BY tickets.agent_id, agents.team
) AS summary_per_agent

SELECT agent_reopen_rates.agent_id, agent_reopen_rates.team, agent_reopen_rates.reopen_rate
FROM agent_reopen_rates
JOIN(
    SELECT team, AVG(reopen_rate) AS average_team
    FROM agent_reopen_rates
    GROUP BY team
) AS average ON agent_reopen_rates.team = average_team
WHERE agent_reopen_rates.reopen_rate=average.average_team;

```

3. CSAT trend by category Write a query returning the average CSAT score per category, per month, for the last 3 months.
```sql
SELECT tickets.category,
      DATE_TRUNC("month", csat_responses.submitted_at)
AS month
      AVG(csat_response.score)
FROM csat_responses
JOIN tickets ON csat_responses.ticket_id = tickets.ticket_id
WHERE csat_responses.subitted_at >= DATE_TRUC("month",CURRENT_DATE) - INTERVAL "3 months"

GROUP BY tickets.category, DATE_TRUNC("month", csat_responses.submitted_at);

```

4. Digging in Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

Before drawing conclusiones, I'd start with the "tickets" table, focusing on the agent_id, ropened_count and priority.  From my experience managing capacity planning for 250+ Data Management professionals and leading a team of CTAs (who manage multiple projects at the same time), I know that a CSAT drop isn't necessarily explained by volume or tenure of staffing, but rather comes from uneven workload distribution  or a silent shift in the type of ticket coming in (even when the total number of tickets remain unchanged).

* WORKLOAD DISTRIBUTION: From opened_at and closed_at I'd look at how many tickets were open simultaneously per agent, not just the total count.
* PRIORITY MIX: I'd want to check whether the proportion of high priority tickets increased this month. If overall first response time stayed flat, but the mix shifted towards more urgent tickets, customers who needed a faster response aren't getting one.
* REOPENED TICKETS: I'd also want to check whether the reopen rate in Billing changed month over month. A ticket that gets reopened usually means the first response didn't fully resolve the issue, which points to resolution quality rather than speed, and wouldn't necessarily show up in volume or staffing numbers.

ADDITIONAL DATA I WOULD REQUEST TO UNDERSTAND MORE:

-Survey response volume per month: If negative responses (1-2 star ratings) increased by 12%, I'd want to know whether that 12% is being driven by a real increase in dissatisfied customers, or simply by a drop in total survey volume this month. For example, if last month we had 100 responses with 5 negative (5%) and this month only 40 people responded, with the same 5 negative (12.5%), the actual number of unhappy customers didn't change, the response pool just got smaller. I'd also check whether we're seeing response bias, as only the most frustrated customers are the ones that respond, while satisfied customers may skip the survey. 

- Ticket subtype: If Billing is a broad category, I want to know whether the drop is coming from one specific subtype (duplicate charges, refund, etc).

- Notes from the low CSAT tickets: the numbers say that something changed, not WHAT changed. I would like to read a sample of 1-2 star tickets to understand the pattern (if it exists).
  
- Process changes: If, in any billing policies, there was a process update, could explain the customer experience shift.

5. Testing a theory Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

Theory 1: The CSAT drop is being driven by a shift in ticket urgency. If the proportion of high-priority Billing tickets increased this month, customers with urgent issues may not be getting the fast response the expect, even if the overall average response time looks unchanged.
```sql
SELECT DATE_TRUNC("month", tickets.opened_at) AS month,
      tickets.priority,
      COUNT(*) AS tickets.count
      AVG(tickets.first_response_minutes) AS avg_first_response

FROM tickets
WHERE ticket.category= "Billing"
AND tickets.opened_at >= DATE_TRUNC("month", CURRENT_DATE)-INTERVAL "6 months"
GROUP BY DATE_TRUNC ("month", tickets.opened_at), tickets.priority
ORDER BY month, tickets.priority;
```

*Following that question 6 points out the solution for question 5 is related to reopen tickets, I will work on that theory*

Theory 2: the drop isn't about response speed, it's about resolution quality. Customers are having to reopen tickets, and that's what is tanking CSAT even though volume and staffing look unchanged.
If reopens spiked in the same month the CSAT dropped, it means that the resolution quality of each ticket might be the cause.


```sql
SELECT DATE_TRUNC("month", tickets.opened_at) AS month,
      AVG(tickets.reopened_count) AS avg_reopens
FROM tickets
WHERE tickets.category= "Billing"
AND tickets.opened_at>= DATE_TRUNC("month",CURRENT_DATE) - INTERVAL "6 months"
GROUP BY DATE_TRUNC("month", tickets.opened_at);

```


7. What you'd actually do Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

If my query confirms that reopens in Billing spiked and cluster around a specific issue type, this would be my week plan:
* Day 1: I'd pull the data and review a sample of 15-20 reopened tickets myself first, just to have enough context going into the conversation and identify some patterns of why the first response is falling short. This not to arrive to conclusions, but so I'm not asking the team to interpret raw data.

* Day 2 and 3: I'd bring my findings to the team in a working session. I'd share what the data shows (reopens are up, concentrated in refund time) and open the conversation for them to provide insight on what they see from their end, experience, etc. As they are the ones actually handling the conversations with customers, they have more context that I can't see from the data alone. I would also provide what I can see only from analyzing the data (patterns), from our team, and let them brainstorm of possible causes, maybe the macro is outdated, or the customers are asking follow-up questions we are not anticipating, etc.

* Day 3: I'd give my manager a heads-up on what we found and what the team and I are planning to adjust. 

* Day 4: Based on that comes out from that conversation, we'd update (or create) a macro or process together, so it reflects what the team actualy know works, not just what I think should work from the outside.
* Day 5: I'd meet 1:1 with agents who had the mot reopens in that issue type to find out whether what we implemented as a team is actually helpful or whether they are running into something more specific like a particular gap in understanding, or a need or more hands-on training on that issue type.

To know if the implementation worked, I'd track Billing's reopened_count and the issue-specific CSAT weekly over the following 3-4 weeks, but not just the team average. I'd specifically monitor the reopen rate for the agents who were struggling most, since an overall average could improve just becase the already-strong agents got even better, while masking the fact that other agents might not be improving. 
If their individual numbers are not moving, that tells me that the root cause wasn't fully addressed and we need to go deeper on an individual level.

8. Reporting up and coaching down You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?

To my manager i'd would say: "The Billing CSAT drop identified traces back to a spike in reopened tickets concentrated around refund timing questions; the team and I worked through it together and updated the macro based on what they were actually running into with customers. We're now tracking the reopen rate weekly, both for the team overall and for the agents who were struggling most, to confirm the fix is landing and I will share the progress update in 2 weeks."

In my 1:1 with the agent: This wouldn't be the first time they're hearing about it as they were already part of the team session where we looked at the data together, so the conversation starts from a shared understanding. I'd asked directly: "Now that we've updated the macro, how's is feeling on your end? Is it giving you what you need, or are you still running into any other type of issue?"
If they're still struggling despite the fix, I want t know specifically where; is it a particular type of refund question that is more complex or is it more about confidence in explaining something to the customer?  Based on that, I'd either walk through a couple of real examples together, or set up more hands-on practice, but the goal is finding out if this a tooling gap we need to close, or the need for more direct mentoring.

The message to my manager is brief and outcome-focused, it is meant to inform, not to explore, so I compressed it to the root cause, action taken and the next checkpoint. The 1:1 conversation is exploratory and specific, because the goal is not to report a result, it is to understand their individual experience and figure out together, whether the fix actually solved their particular gap or something more targeted is still needed.
