This file is a general overview of my agentic teaching repository for SQL.

The main goal of this repo is to cooperate with my LLM on daily basis to serve as my teacher. 
The agent will generate valid questions about my database, that will require me to write and run SQL queries and therefore practice SQL to maintain my intermediate SQL level (as right now I'm holding an Intermediate SQL cert@Hackerrank) + pass the advanced SQL certification soon (I didn't try yet, but I honestly feel like I did not master all of the advanced concepts, like ALL OF THE WINDOW FUNCTIONS, etc.). The agent will also provide me with feedback (whenever possible) so that I can improve my SQL gradually.

The agent's main instruction is in CLAUDE.md (hidden from the public view).

As for data and the environment for running the actual SQL queries, I am using a local Postgres db (you can find the database schema in schema.md (contains tables + data examples)).

Files:
- schema.md - database schema - tables + data examples
- question_examples.md - 10 examples of good questions (currently - 4.12) + code snippets that would be used to find the answers (POTENTIAL ANSWERS)

TO DO:

- Instructions for the agent
- A file structure which will contain feedback/tasks archive and current tasks - I'm aiming for brevity and clarity here (memory saving)