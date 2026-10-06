# Eric-G173.github.io

## Code Review:
<iframe width="800" height="450" src="https://www.youtube.com/embed/cIAx5uD1VyU" title="Eric Grana Code Review" frameborder="0" allowfullscreen></iframe>

<br>

## Artifact One: 
### Enhanced Code:
[CS499 Artifact One_Enhanced (1).zip](https://github.com/user-attachments/files/33123694/CS499.Artifact.One_Enhanced.1.zip)

### Original Code:
[Original_Code.zip](https://github.com/user-attachments/files/33123736/Original_Code.zip)

### Explanation:
**Artifact Description**
<br>
The artifact being modified here is from my CS 340 (Client/Server Development) class, and this would be the second project of that class. The project in specific is an SPA dashboard that is being used to display rescue shelter dogs and where they were found. It has different information such as a pie chart comparing how many breeds were found, a map to show the location of the dog getting found, and a spreadsheet style showcase of all the animals including their name, gender, etc. Filters are also attached so you can filter the dogs by a general type such as mountain dogs or winter dogs. This comes together to allow for rescue members to have an easy time tracking dogs that were found, and are capable of seeing data trends such as what types of dogs are getting rescued the most. The data the artifact uses comes from a mongo database, which has an admin account created so we can properly do CRUD operations with it. This artifact was created around August of 2026, so it is using recent updates and technology for this assignment.
<br>
**Inclusion of Artifact Justification**
<br>
The reason I selected this artifact is because it was a complex and strong project that used many different aspects of computer science, but left some flaws. While the project worked fine in a jupyter codio environment, using it outside of that would completely break the project and render it useless. Also looking at the code itself, a lot of the comments are basic educational comments from the original assignment, and there are no security features adapted. I felt like this was the perfect chance to take this project and transform it from a school project to a fully functioning dashboard that is independent of jupyter or other environments people would not have access to. I plan to showcase my sense of system design with the changes, as well as security. The artifact has been improved in multiple ways. To start with design, I was able to successfully remove all ties to jupyter and the codio environment, and made it so the project is functioning within its own local environment. This was huge because this project originally would not run outside of codio. I then used security strategies and hid all the database and admin information behind a .env, that way if deployed, people would not be able to steal these credentials. Better comments were also added, and the old educational assignment comments were removed to make the project look more professional. Finally, I decided to break up the project into different component files, that way viewers can have an easier time understanding where each component is and how it can be modified. 
<br>
**Meet Course Outcomes**
<br>
I do believe I met the course outcomes laid out for this project. The goal was to take my skills I have learned throughout the degree and enhance the project in unique ways. I think I did that well with the removal of jupyter, as well as general design changes being made. I was able to take this project from unprofessional to a professional work that would be better understanded and looked at by software professionals. I so far do not have any outcome-coverage updates, the main update I did make was removing the jupyter as I did not plan on doing that originally, but it turns out I had to in order to make the project work. Other than that, every enhancement I made was originally outlined in my goals for this project.
<br>
**What I Learned as I was Creating**
<br>
The main ideas I learned from creating this project was how complex system design can be, and how much planning you have to do beforehand when going into adding components and modifying code. The security part was easy and straightforward, but I was stumped a bit on how I would go about splitting the file into components. I did not want to break every single piece of code down or else the artifact would look cluttered. I had to draw a line between what would help make the project more readable and also not being too complex and having too many files. I think I was able to overcome this situation well, and with the right planning was successfully completed.

