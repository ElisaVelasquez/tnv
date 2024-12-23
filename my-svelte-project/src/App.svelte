<script lang="ts">
  import { onMount } from "svelte";
  //import { goto } from '$app/navigation';
  import { tick } from "svelte";    
  import { onDestroy } from 'svelte';

  let currentQuestionIndex = 0;
  let typingText = "";
  let showQuestions: any = [];  // Keeps track of shown questions and inputs
  let showFirstPage = true;
  let showNextQuestion = false;
  let questionBeingTyped = "";
  let showParagraph = false;
  let currentTime:  string | Date | null |undefined;

  let interval: any;
  let questions = [""]; 
    
  // Reactive block to update CreditQuestion and questions array
  $: {
        questions = [
            "What is your citizenship?",
            "Do you have a diploma or degreee",
            "Dou you have a criminal history such as being convicted of any felonies?",
            "Do you have any education or work experience that qualifies under NAFTA occupations?",
            "Are you currently outside of the US?",
            "Are you planning to apply inside or outside of the US?",
        ];
    }

   // This function allow you to insert answers into the answers table
  /* async function setAnswers(user:User) {
        const {data: profileData, error} = await supabase
                .from("answers")
                .select("*")
                .eq('profile_id', user.id)

        if(!error){
            if(profileData[0]){
                userInput.firstAnswer = profileData[0].question_1;
                userInput.secondAnswer = profileData[0].question_2;
                userInput.thirdAnswer = profileData[0].question_3;
                userInput.dedicated_credits = profileData[0].dedicated_credits;
            }
        }
    } */

    // Function to observe changes in the input field
    function monitorInputValueChanges() {
        // Locate the container holding the input field
        const cardCvcContainer = document.getElementById('card-cvc-element');
        if (cardCvcContainer) {
            // Find the input field inside the container
            const inputElement = cardCvcContainer.querySelectorAll('input')[0];
            if (inputElement) {
                // Attach an event listener to monitor changes
                inputElement.addEventListener('input', (event: Event) => {
                    const target = event.target as HTMLInputElement;
                    console.log(`Input value changed: ${target.value}`);
                });
                console.log('Input monitoring started.');
            } else {
                console.error('Input element not found inside #card-cvc-element');
            }
        } else {
            console.error('#card-cvc-element not found');
        }
    }

    /*
    onMount(async() => {
        monitorInputValueChanges();
        updateTerminalImageSrc();
        const {data} = await supabase.auth.getSession();
        user = data.session?.user || null;
        if (user){
            await setBalence(user);
            if(balance == 0){
                goto("/payment");
            }
            await setAnswers(user);
            await getCurrentUTCTime(user);
            await setDedicated(user);
            // After fetching the data, check and start the countdown if needed
            await handleFetchedDataCountdown();
        }else{
            goto('/');
        }
        const allAnswered = Object.values(userInput).every(answer => {
            if (typeof answer === 'string') {
                return answer.trim() !== '';
            } else if (typeof answer === 'number') {
                return answer !== null && answer !== undefined;  // Adjust based on your expected conditions
            }
            return false;
        });
        showFirstPage = !allAnswered; // Show form if any answers are missing
        if (!showFirstPage) {
            populateAnswers();  // If already answered, populate the shown questions
        }
    });

    function populateAnswers() {
        showQuestions = questions.map((question, index) => ({
            question,
            answer: index === 0 ? userInput.firstAnswer 
                : index === 1 ? userInput.secondAnswer 
                : index === 2 ? userInput.thirdAnswer
                : userInput.dedicated_credits
        }));
    }

    // Function for the typing animation (for when the form is displayed)
    function typeNextQuestion() {
        typingText = "";  
        const fullText = questions[currentQuestionIndex];
        questionBeingTyped = fullText;
        let i = 0;

        const typingInterval = setInterval(() => {
            typingText += fullText.charAt(i);
            i++;
            if (i === fullText.length) {
                clearInterval(typingInterval);
                showNextQuestion = true;
                showQuestions.push({ question: typingText, answer: '' });
                typingText = '';
            }
        }, 25);
    }

    // Show the first question on mount, if applicable
    $: {
        if (showFirstPage && showQuestions.length === 0) {
            typeNextQuestion();
        }
    }
   
    // Handle pressing 'Enter' or clicking to move to the next question
    async function handleNext() {
        showNextQuestion = false;
        showQuestions[currentQuestionIndex].answer = currentQuestionIndex === 0 ? userInput.firstAnswer 
            : currentQuestionIndex === 1 ? userInput.secondAnswer 
            : userInput.thirdAnswer;

        currentQuestionIndex++;
        if (currentQuestionIndex < questions.length) {
                typeNextQuestion();
        } else {
            await saveAnswers();  // Save answers to the database when all questions are answered
            goToNextPage();
        }
    }

    function autoAdvanceInput(nextInputId: string) {
        // Use tick() to wait for the DOM to finish rendering before trying to access the input
        tick().then(() => {
            const input = document.getElementById(nextInputId) as HTMLInputElement;
            console.log(input); // To check if the input is found
            if (input) {
                input.focus();  // Automatically focus the input when it is rendered
            } else {
                console.error("Input element not found");
            }
        });
    }
        // Your reactive block to handle focusing on the inputs
    $: {
        if (showNextQuestion && currentQuestionIndex) {
            const inputId = `input${currentQuestionIndex + 1}`; // Input IDs start from 1 (e.g., input1, input2, etc.)
            autoAdvanceInput(inputId); // Automatically focus the next input field
        }
    }
    // Function to save or update the answers in Supabase
    async function saveAnswers() {
        if (user) {
            const { error } = await supabase
                .from("answers")
                .insert({
                    profile_id: user.id,
                    question_1: userInput.firstAnswer,
                    question_2: userInput.secondAnswer,
                    question_3: userInput.thirdAnswer,
                    dedicated_credits: userInput.dedicated_credits
                })

            if (error) {
                console.error("Error saving answers:", error);
            } else {
                console.log("Answers saved successfully");
                // saving the dedicated credit as well 
            }
        }
    }

    async function createTransaction(credit:number) {
        if (user) {
            const { error } = await supabase
                .from("transactions")
                .insert({
                    profile_id: user.id,
                    amount: credit,
                })

            if (error) {
                console.error("Error saving Transaction:", error);
            } else {
                console.log("Transaction saved successfully");
            }
        }
    }    

    async function updateAnswers() {
        if (user) {
            const { error } = await supabase
                .from("answers")
                .update({
                    question_1: userInput.firstAnswer,
                    question_2: userInput.secondAnswer,
                    question_3: userInput.thirdAnswer
                })
                .eq('profile_id', user.id)
            if (error) {
                console.error("Error updating answers:", error);
            } else {
                console.log("Answers updated successfully");
            }
        }
    }    

    async function updateBalence(newBalence:number) {
        if (user) {
            const { error } = await supabase
                .from("balences")
                .update({credit_balance:newBalence})
                .eq("profile_id", user.id)
            if (error) {
                console.error("Error updating balences:", error);
            } else {
                console.log("balences updated successfully");
            }
        }
    }

    async function updateDedicatedCredits(credits:number) {
        if (user) {
            const { error } = await supabase
                .from("answers")
                .update({dedicated_credits:credits})
                .eq("profile_id", user.id)
            if (error) {
                console.error("Error updating dedicated credit:", error);
            } else {
                console.log("Dedicated credit updated successfully");
            }
        }
    }    

    function goToNextPage() {
        showFirstPage = false;
    }

    // Function to start the countdown and validate credits
    async function startCountdown() {
        showPaymentWindow = false;
        showParagraph = false;
        console.log("Im the one : startCountdown");
        const credits = typeof creditsToDedicate === 'string' ? parseFloat(creditsToDedicate) : creditsToDedicate;

        if (balance == 0 || credits > balance) {
            showPaymentWindow = true;
            countdown = countdownOfBadTransaction;
            currentTime = await getCurrentUTCTime(user);
            clearInterval(interval); // Clear any previous intervals
            interval = setInterval(() => {
                countdown--;
                if (countdown <= 0) {
                    clearInterval(interval);
                    goto("/payment");
                }
            }, 1000);
        } else {
            balance -= credits;
            await createTransaction(credits);
            await updateBalence(balance);
            await updateDedicatedCredits(parseInt(userInput.dedicated_credits));
            showParagraph = true;
            console.log("Im the one : startCountdown");
            currentTime = await getCurrentUTCTime(user);
            countdown = countdownOfSuccessfulTransaction;
            clearInterval(interval); // Clear any previous intervals
            interval = setInterval(() => {
                countdown--;
                if (countdown <= 0) {
                    clearInterval(interval);
                }
            }, 1000);
        }
    }

    // Clean up interval when the component is destroyed
    onDestroy(() => {
        clearInterval(interval);
    });

    
    function handleKeydown(event:any) {
        if (event.key === 'Enter') {
            event.preventDefault();  // Prevent the form from submitting
            handleNext();            // Call your existing handleNext function
        }
    }
    $:console.log(showQuestions);
/*     function editAnswer(index: number) {
        if (showQuestions.length >4){
            showQuestions.pop();
        }
        if (index == 1){
            showQuestions[2].isEditing = true;  // Set the editing mode for the selected question
        }else if(index == 2){
            showQuestions[1].isEditing = true;
        }else if(index == 3){
            showQuestions[0].isEditing = true;
        } 
    } */
     function editAnswer(index: number) {
        if (showQuestions.length >4){
            showQuestions.pop();
        }
        showQuestions[index].isEditing = true;  // Set the editing mode for the selected question
    }

</script>

<html lang="en">
	<head>
		<meta charset="utf-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1" />
		
	</head>
	<body>
        <p>Hola</p>
	</body>
</html>


