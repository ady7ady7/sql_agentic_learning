To keep it simple, this file presents a concise overview of my agentic teaching repository for SQL.
-----------------------------------
The agent will generate questions about my database, that will require me to practice SQL queries to maintain my intermediate SQL level (certified as Intermediate on Hackerrank, @4.12.25) + hopefully allow me to pass the advanced SQL certification after 8-12 weeks of daily practising.
------------------------------------
The agent's main instructions are in CLAUDE.md (hidden from the public view).

ENVIRONMENT

As for data and the environment for running the actual SQL queries, I am using a local Postgres db (you can find the database schema in schema.md (contains tables + data examples)). If necessary, I can also share my screen to show the actual database.

FILES

- schema.md - database schema - tables + data examples

- question_examples.md - 10 examples of good questions as an inspiration for the agent + code snippets that would be used to find the answers (POTENTIAL ANSWERS)

- tasks.md / tasks_archive.md - the first file holds the present tasks (for the current day); after we're finished with a given day, they're moved to the tasks_archive.md, so that we won't clutter the agent's memory, and it's simply cleaner

- feedback.md / feedback_archive.md - same idea as anove, but it's worth adding that the feedback is bidirectional - I will rate the agent for the tasks, give feedback and assess the logic and the difficulty level of the tasks; the agent will obviously give me constructive feedback in terms of my SQL proficiency and best practices

- good_q_examples/bad_q_examples - I thought it might be a good way of introducing RAG to our agent's work, as based on my previous experience in agentic learning of Python, this might be useful for the agent to check sometimes