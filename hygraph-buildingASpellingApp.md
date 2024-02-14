# Article Title: Building a Spelling App

## The Why: What is the Need?
I am a parent and I also work in a school. I am always trying to figure out a way to automate processes or just make my life easier. When my child brought home a list of spelling words that they were going to be tested on, I thought there had to be an easy-to-use site for this. Then, I realized that I could just build the thing rather than find the thing. So, let’s do this together. What you’re going to do is this: build a website that has a simple form with a button, the button will use a text-to-speech library reading from a word list, and then the form will check if what is typed matches the current word in the list. The word list will be maintained in a headless CMS so that the data isn’t contained in the website itself. Let’s get started!

**Disclaimer: This guide is not a step-by-step guide. It is recommended that you understand some basics about React, Javascript, and GraphQL queries before starting.*

## Getting Set Up: Where Do I Start?
Before we get started, let’s take a sneak peek at what you’re going to build. The site lives at [CDS Spells!](https://cds-spelling-game.netlify.app/) where you can edit the word list and create a new quiz. I built this site using NextJS and TailwindCSS and I use Hygraph for data management and Vercel for hosting the site.

Start by opening up your terminal and using npm to build your project. Run the command `npx create-next-app@latest` to build your project. This will guide you through all of the build steps. In addition, you will want to install 2 libraries with npm. Install both `graphql-request` and `react-speech-kit`. Once you get the project created, go ahead and open up your project folder with your favorite text editor. NextJS will give you a good boilerplate, but you will want to get rid of most of that.

Since we are using the new App router in NextJS 14, you will want to do a few things. The app is going to have a landing page with a link to the quiz route. 
```
<p>
  <Link href="/quiz">Take a Quiz</Link>
</p>
```

To add a new route, you will create a folder in the app directory and name it ‘quiz’. Then in this new folder add a new page and save it as page.js. You can test your new route if you add a default export where it returns a message like “Quiz” between some h1 tags. 
```
export default async function Quiz() {
  return (
    <h1>Quiz Page</h1>
   )}
```

## Data Set Up: Using Hygraph and GraphQL
Now that you have a basic app with a new route added, you’re ready to start pulling data. There are a few ways to pull data into a NextJS app. In this app, you are going to be using Hygraph to pull in GraphQL data. [Hygraph](https://hygraph.com/) is a headless CMS that makes creating, mutating, and reading data pretty easy. If you haven’t already, go ahead and sign up with an account. Once you get your account made, you’ll have the option to create a project.

Once you have created your project, you will create a model for your project. You can add what fields you want to add, but these are the fields that I used for my project:
- userName - this will be the user that created the quiz
- quizName - this will be the name of the quiz that the user set
- words - this will be the list of words
- slug - this will be the link for the quiz as a combination of the userName and the quizName
The only field of these shown above that will be unique is the slug. The slug needs to be unique so that duplicate links are not created for different quizzes. Using the menu, click the Schema link to add a Model to your project. 

Now that you have a model set up, you’re ready to set up a test quiz and see if you can pull the data in. Entries can be added by clicking the Content option. You can also add content through API endpoints, but we’ll get to that later. Create a new entry in your content. For my app, I used a dedicated character for separating my words in the wordlist like a semi-colon. Feel free to think about how you want to separate words in your wordlist. Once you get your information typed in, click the save and publish button. Publish will take it out of the draft stage and put it on the published stage. The published stage is the default for the public API endpoint for reading your data.

Find the Project Settings menu now. Then go down to API Access. When you first set up a project, nothing is shared. In the Public Content API, you can turn on defaults for reading the data. If you’re planning to use data that needs to be secured with an access token, you can do that on the menu item below this. We’re not going to publish the API endpoint, so I feel public access is going to be fine for a list of spelling words. Then, you’re going to grab the URL for your API endpoint. It is going to be the top link of this page under the heading Content API. That link will be what you fetch from your app. If you haven’t already, create a .env.local file in the root directory of your project. You will create a variable for your endpoint there:
```
# .env.local
GRAPHQL_API_URL = "https://url-for-my-api"
```
Now, you’re ready to fetch some data in your app!

In the quiz page that you created, you’re going to add the following code:
```
// ./app/quiz/page.js
const hygraph = new GraphQLClient(process.env.GRAPHQL_API_URL)
const { wordLists } = await hygraph.request(
{
    	wordLists {
        	id
        	slug
        	userName
        	words
      	}
})
```
This will pull your word list entries from Hygraph (all 1 of them) and assign the data to the destructured object wordLists. Then, you can map over the word lists that have been imported by their id number in your return statement for the default export. 
```
{wordLists.map((wordList) => {
        	return (
            	<div key={wordList.id}>
                	<h2>{wordList.userName}</h2>
                	<p>{wordList.words}</p>
            	</div>
        	)
    	})}
```
Get your project running locally and navigate to the /quiz route to see if the data that you entered in Hygraph is showing up. Huzzah! You are fetching data from your headless CMS with your API endpoint.

## Basic App Flow
Now that you have connected your app to your Public API Endpoint, you can start building out the logic for the app. The app should work this way:

1. Load the route for a quiz.
2. Pull the data from Hygraph asynchronous.
3. Set the current word as a random word from the list.
4. Allow for the user of the app to hear the word with a ‘speak’ button.
5. Allow for the user to to type the word in a text field and give the user feedback if it's correct or not.
6. Repeat for more words in the list.

Let’s start with loading the route. You can use NextJS dynamic routing with a request to your content in Hygraph to build out our routes for each quiz. In your app, create a folder and name it quizzes. Then inside of that folder, create another folder and name it [slug]. Then inside of that folder create a page.js file. This will create routes with the syntax: `<hosting url>/quizzes/<slug from hygraph>`. Your new page inside the [slug] folder will be a template for all quizzes. To ensure all data gets to the right places, you’ll do 2 different GraphQL requests: one to generate the dynamic routes and one for the content that needs to be on each route. The one to generate the routes uses `generateStaticParams()` from NextJS:
```
export async function generateStaticParams() {
    const hygraph = new GraphQLClient(process.env.GRAPHQL_API_URL)
    const { wordLists } = await hygraph.request(
    {
    	wordLists {
        	    slug
      	}
    })
         return wordLists.map((wordList) => ({
    	slug: wordList.slug,
          }))
  }
```
With this on the page, each quiz that has a slug entry will be generated on the fly with dynamic routing. When new quizzes get added, they will automatically get routes created for them when this request gets made. The second request that will be made is similar to one that was created in the previous step. Just copy and paste that GraphQL request over and update it to filter your request by the slug: wordLists (where: {slug: "${params.slug}"}) {.

Now that routes are set up and we know that the words from each word list are being routed to the right route, it's time for the game flow. You’ll take in the words as a String and splice it with punctuation marks. If the word list is separated by commas, then splice by commas. Following that, trim each word’s whitespace and shift it to lowercase. Then, you’ll want to randomize the order of the words in your array so that it’s not the same every time you come to the page. I did this by iterating over the list swapping random terms:
```
for(let j = wordArray.length - 1; j > 0; j--) {
	const k = Math.floor(Math.random() * (j + 1));
	[wordArray[j], wordArray[k]] = [wordArray[k], wordArray[j]];
 }
```

At this point, you have an array of words that are in random order. The app will iterate over this list to assign the current word to an element in that array. This can be done using React’s useState. The next step is to add the user input. For this app, you should build out a simple UI that includes:
- a ‘speak’ button that will speak the chosen word
- a text entry on a form
- a submit button for the form
- a ‘next’ button for the user that will skip to the next word
In addition to initializing the current word that needs to be guessed, a few more items need to be taken care of:
- create a score variable 
- create a variable for the guessed or typed word
- initialize the speak object using useSpeechSynthesis()
The library that you will use is from the library, react-speech-kit. You can find the documentation for react-speech-kit in [npm’s library](https://www.npmjs.com/package/react-speech-kit). To get all of this set up, add this code after your code segment to pick a word.
```
  const [word, setWord] = React.useState(wordArray[0]);
  const [guessedWord, setGuessedWord] = React.useState('');
  const [score, setScore] = React.useState(0);
  const { speak } = useSpeechSynthesis();
```
Now with this setup, you can tie the ‘speak’ button with an onclick event to call the speak function. 
```
<button onClick={() => {
speak({ text: word  });
}}
>Speak</button>
```
After the user uses the ‘speak’ button, they will type into the form with the text box that you created already. The submit button will then compare what they typed to what the word was set to. Following this, you can give the user feedback on whether they were correct or not. The easiest way to do this would be with alert messages. You can also use a banner that has absolute positioning with a z-index higher than your main content. If they get it right, add 10 points to their score and move on to another word. 

## Customization: Add Some Flair
Now with what we’ve done so far, it's just the workflow of the program. Here are some ideas to make it more attractive and engaging:
Make the background not a single color - in my final version I created a radial background that was a combination of the school colors for the students that were going to use this
Use additional features from the react-speech-kit library like the pitch and speed options
Create fun messages for students - use emojis!
Add in CSS to make it a mobile version so that it can be used from any device

## Adding More: The More Robust Version
Moving from a prototype to finished product involves a few more steps. In my version, I changed the organization of the program. I did this because I wanted users to be able to login and create their own quizzes. By using things like [nextAuth](https://next-auth.js.org/) or [Clerk](https://clerk.com/), you can link Google SSO to your app. Then, your app can have a profile page for users when they login. The profile page can have a form where they can create a new quiz and then populate the area under the form with their current quizzes. By taking advantage of NextJS’s revalidate parameter in its fetch API, you can let NextJS update the page by comparing what’s in the cache and fetching new data if the current list of quizzes has gone stale.

The quizzes can have a better point system. In the point system that is listed above, when a student hits refresh or leaves the site and comes back, their points will be reset to zero. The teacher then doesn’t get any feedback. By adding a points column to the schema for your quizzes in Hygraph, you can keep track of how many cumulative points 
have been earned for the quiz. This makes each quiz a community effort and doesn’t reward specific students. It can bring a piece of data to teachers that represents how much work a class has put into learning a list of words. 

The live version of this project lives at [CDS Spells!](https://cds-spelling.netlify.app/) and the repo can be found on [Github](https://github.com/ryanjames1729/cds-spelling).