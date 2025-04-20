## Crafting an AI Prompt App with Scrum: A 6-Week Coding Adventure 

### Introduction
[Chingu](https://chingu.io) is an online platform that brings together Scrum Masters and developers from around the world to participate in a collaborative six-week voyage. During this time, teams work together to deliver an app, all while adhering to Scrum principles.   
Our recent project was the creation of an AI prompt app, which we built using agile methodology and teamwork. 

### Starting the Voyage
We were a team of seven, consisting of five developers, a Scrum Master, and a Shadow Scrum Master. To kick off our voyage, we began brainstorming ideas for a project we could complete within six weeks. After discussing various options, we decided to create an AI Prompt App that integrates with the Google Gemini API.  
This app features a pentagram-style form, allowing users to create and submit custom queries to the AI model by filling in five key components:

    Persona: Defines the role or identity of the AI for context.

    Context: Describes the background or scenario in which the AI should operate.

    Task: Specifies the task or action the AI should perform.

    Output: Details the desired outcome or format for the AI's response.

    Constraint: Sets any limitations or boundaries for the AI’s response

With a clear concept in place, we moved on to defining a simple yet functional Minimum Viable Product (MVP). Estimating development time at this stage was challenging, so we kept the scope minimal and allocated a week for bug fixes. To stay user-focused, one team member created a Figma flow illustrating the app’s pages and how they should behave based on user login status

From this, the scrum masters created a Jira board with a backlog of all tasks to be completed for our MVP (and some stretch goals). We agreed on a tech stack (see below), and agreed on some ground rules (such as daily standups, a GitHub workflow etc.).

![JIRA board](/images/jira_board.PNG)



### Tech Stack
- **Backend**: NodeJS, Express, TypeScript
- **Frontend**: React, Tailwind CSS, TypeScript
- **Database**: PostgreSQL (hosted on Supabase)
- **Deployment & Hosting**: Nginx, Render, Netlify
- **Collaboration**: Jira
- **Prototyping**: Figma


## Development Experience
As mentioned above, we were a team of seven, all from different backgrounds and all bringing different skills / perspectives to the voyage.

We decided on a 3 meetings a week approach (with daily standups inbetween), in theory, we'd have one day to introduce the sprint (with each sprint being 5 days long), add items from the backlog to the sprint and assign developers to those tasks. Then, we would have a meeting mid-week to discuss any challenges that came up and then finally on a Friday, one quick meeting to close off the sprint and hold a sprint retrospective.

At times, it was helpful for team members to divide into unofficial "sub-groups" to work collaboratively on a feature. When this happened, it allowed bugs to be addressed early as well as improving the coding practices early on.

On the technical side of things, it was my first time using NodeJS + Express + TypeScript as well as PostgreSQL. While NodeJS + Express was a bit of learning curve for me, I enjoyed using them. On the other hand, I found PostgreSQL quite easy as I was already familiar with MySQL, and TypeScript was reasonably straightforward.   
We created a REST API for the frontend to communicate with the backend and vice versa and secured the endpoints with JWT based authentication.

At the end of the voyage, I am proud to say that we collectively achieved our MVP and deployed the fully working application to the web.



### Git Best Practices
We strove to adhere to Git best practices in this voyage. Our workflow consisted of feature branches, hotfix branches, a development branch and a main branch.   
We disallowed direct pushes to the development or main branch in order to isolate any bugs that occured in the feature branch were they occured. When a developer was satisfied their feature was complete, they created a Pull Request (PR) to merge their feature branch into the development branch, which another developer would have to approve (or request changes) before merging the changes. Similarly, at the end of a sprint a PR would be created to merge the development branch into the main branch.



## Testing
Given the time constraints, testing on this project was limited to testing API reponses through Postman as well as having developers manually test each other’s PRs.



## Deployment
For deployment, the goal was to have a completely free deployment that didn't require a credit card. We decided on Netlify + Render.

Render has "cold-starts" whereby the user may experience a delay if the app hasn't been used recently. This is a limitation of the free tier but it was a tradeoff we were willing to accept as this app is mainly a proof of concept.

One of the challenges was coming up with a hosted database that didn't have any excessive restrictions (e.g. Render automatically deletes the database 30 days after creation if you use the free tier.)  
In the interests of saving time, we decided to initially use a local PostgreSQL database while we researched what hosted database to use, knowing that when we eventually decided on what hosted DB to use, the code changes would be miminal.

Eventually, we decided on a Supabase hosted DB, which was very easy to setup.

## Deployed APP
[Press here to access the AskIQ App](https://askiq-live.netlify.app/)  

![ASK_IQ_DEMO](/images/askiq_demo.PNG)  


## Lessons Learnt
- Importance of daily standups / communication so issues can be addressed quickly and "head-on"  

- Avoiding "scope change" (e.g. don't deviate from the MVP at late stages during a sprint)

- Keeping meetings action based and focus on achieving the MVP  

- Avoid over-complicatating the organisational side of the project (e.g. for this small project, Trello might of been simpler than Jira)

## Future Enhancements
- Allow users to save personas and load them subsequently

- Add rate limits to the app  

- Add unit and integration testing  

- Minimise the "cold-starts" on Render

## Conclusion
Working on this project with the team was an excellent opportunity to gain experience in a simulated software development environment.   
We aimed to mimic a real-world software project by following Scrum principles and maintaining a solid GitHub workflow. Although we faced challenges along the way, we were able to deliver our MVP, deploying the completed app to the web.