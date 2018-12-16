# Title
A time story of offshore societies.

# Abstract
The dataset we chose for this project is the Panama Papers. Our main idea is to visualize links between people and offshore societies, the origin location of these people as well as the location of these offshore societies. On one side we want to know what is the proportion of companies created from each origin country. And on the other side we are interested in the proportion of societies created in each country. We want to visualize this data across the years to understand how tax evasion evolves. Our aim is to understand how the numbers of these offshore companies (with respect to the location and origin of people creating them) have evolved across time see if the issue is becoming worse with time. 

# Research questions
1) How does tax evasion evolves over time? Does the number of offshore societies increases? 

2) Do people that are related to one offshore account tend to be related to many more? In other words, is it more likely to have another account once you already have one, than it is to have at least one account?

3) Do many people share one offshore society or do they tend to have their own?

(OLD) 4) How many intermediate societies are there between people and their offshore company? 
After working more with the data we realized this question didn'ts really made sense. Entities connected together are not intermediaries, they usually are similar entites or have the same name. Here is the new question we replaced it with :
(NEW) 4) How does the proportion of intermediaries depend on the category of company (number of connections) ?

5) Is there a correlation between the location of the people and the location of their offshore society?

# Dataset
We used the Panama Papers dataset. The dataset is organized as a graph, split across multiple csv files. Nodes represent entities, such as people, intermediaries, offshore societies and addresses. While the edges represent links between all these entities. We wish to use all this data (352MB) to recreate a database that will be adapted to our research questions.

# A list of internal milestones up until project milestone 2
We've decided to set ourselves three internal milestones, one each week up to project milestone 2 :
* 11/11 : load, clean and preprocess the data
* 18/11 : create new dataframes that correspond to the format we need to answer our questions
* 25/11 : properly comment the code and create a structured plan for the rest (up to the presentation)

# A list of internal milestones up until project milestone 3
Again, we set ourselves three internal milestones, one each week up to project milestone 3 :
* 2/12 : Respond to research questions 1 and 2 
* 9/12 : Respond to research questions 3,4 and 5
* 16/12 : Finalize the notebook and write the report

Decided to remove 'other' node type, because it's only ~3K nodes, and they are all isolated, it just brings noise.

Merged Node/Edge dataframe into one big one (edges_completed).

# Questions for TAs
If we have some questions, we're going to ask TAs in person during labs and office hours :)

# Datastory
We decided to present our result in the form of a data story. It is available at : https://yymartin.github.io/ADA_Project_Cobra/

# Contribution of group members
All members contributed equally to the project. But specifically the members focused on :
* Simon : preprocessing of the data to allow accessing it easily through dataframes, statistics on the data, research question 4, speaking during final presentation
* Yoan : creating maps, data story setup, research question 1 and 5, poster for final presentation
* Pierre : creating the visualizations and displaying the results, writing report, research questions 2 and 3, poster for final presentation