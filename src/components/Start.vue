<template>
  <div class="app-container">
    <!-- Progress Bar -->
    <div v-if="currentStep === 'survey'" class="progress-bar">
      <div class="progress" :style="{ width: `${progress}%` }"></div>
    </div>

    <div class="content-container">
      <!-- Enqueteur Input Step -->
      <div v-if="currentStep === 'enqueteur'">
        <h2>Prénom enqueteur :</h2>
        <input class="form-control" type="text" v-model="enqueteur" />
        <button
          v-if="enqueteur && !isEnqueteurSaved"
          @click="setEnqueteur"
          class="btn-next"
        >
          Suivant
        </button>
      </div>

      <div v-if="currentStep === 'poste'">
        <h2>Zone de l'enquête</h2>
        <!-- New PDF button for Poste -->
        <button @click="pdfToShow = '/PASO.pdf'" class="btn-pdf">
          Voir le plan des postes
        </button>
        <button
          v-for="(option, index) in firstQuestion.options"
          :key="index"
          @click="selectPoste(option)"
          class="btn-option"
        >
          {{ option.text }}
        </button>
      </div>

      <!-- Start Survey Step -->
      <div v-else-if="currentStep === 'start'" class="start-survey-container">
        <h2>
          Bonjour,<br />
          pour mieux connaître les usagers de la gare de Corbeil, Ile de France
          Mobilités <br />
          souhaiterait en savoir plus sur votre déplacement en cours.
          Auriez-vous quelques secondes à nous accorder ?»
        </h2>
        <h2></h2>
        <button @click="startSurvey" class="btn-next">
          COMMENCER QUESTIONNAIRE
        </button>
      </div>

      <!-- Survey Questions Step -->
      <div v-else-if="currentStep === 'survey' && !isSurveyComplete">
        <div class="question-container" v-if="currentQuestion">
          <h2>{{ currentQuestion.text }}</h2>

          <!-- PDF Button for QNV2A and QP3A -->
          <button
            v-if="['Localisation_14a_ModeO', 'Localisation_15_ModeO','18a_O', 'Localisation_14a_ModeD', 'Localisation_15_ModeD', '18a_D'].includes(currentQuestion.id)"
            @click="pdfToShow = '/Plan.pdf'"
            class="btn-pdf"
          >
            Voir le plan du parking
          </button>
          <!-- PDF Button for Q10 -->
          <button
            v-if="currentQuestion.id === 'Q10'"
            @click="pdfToShow = '/PASO.pdf'"
            class="btn-pdf"
          >
            Voir le plan des postes
          </button>
          <!-- Standard options -->
          <div
            v-if="
              !currentQuestion.freeText && !currentQuestion.usesCommuneSelector && !currentQuestion.multipleChoice
            "
          >
            <div v-for="(option, index) in currentQuestionOptions" :key="index">
              <button
                v-if="!option.hidden"
                @click="selectAnswer(option)"
                class="btn-option"
              >
                {{ option.text }}
              </button>
            </div>
          </div>

          <!-- Multiple Choice Options -->
          <div v-if="currentQuestion.multipleChoice">
            <div v-for="(option, index) in currentQuestionOptions" :key="index">
              <button
                v-if="!option.hidden"
                @click="toggleMultiSelect(option)"
                :class="['btn-option', { selected: isOptionSelected(option) }]"
              >
                {{ option.text }}
              </button>
            </div>
            <button
              @click="confirmAndNextQuestion"
              class="btn-next"
              :disabled="!(answers[currentQuestion.id] && answers[currentQuestion.id].length > 0)"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>

          <!-- Commune Selector -->
          <div v-if="currentQuestion.usesCommuneSelector">
            <CommuneSelector
              v-model="communeSelections[currentQuestion.id]"
              v-model:postalCodePrefix="postalCodePrefixes[currentQuestion.id]"
            />
            <p>
              Commune sélectionnée ou saisie:
              {{ communeSelections[currentQuestion.id] }}
            </p>
            <button
              @click="handleCommuneSelection"
              class="btn-next"
              :disabled="!communeSelections[currentQuestion.id]?.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Street Selector (new) -->
          <div v-if="currentQuestion.usesStreetSelector">
            <StreetSelector v-model="streetSelections[currentQuestion.id]" />
            <p>
              Rue sélectionnée ou saisie:
              {{ streetSelections[currentQuestion.id] }}
            </p>
            <button
              @click="handleStreetSelection"
              class="btn-next"
              :disabled="!streetSelections[currentQuestion.id]?.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <div v-if="currentQuestion.usesGareSelector">
            <GareSelector v-model="gareSelections[currentQuestion.id]" />
            <button
              @click="handleGareSelection"
              class="btn-next"
              :disabled="!gareSelections[currentQuestion.id]"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Free Text Questions -->
          <div v-if="currentQuestion.freeText">
            <div class="input-container">
              <input
                v-if="['Q3', 'Quai', 'TempO', 'TempD', '19aD', '19aO'].includes(currentQuestion.id)"
                v-model="freeTextAnswer"
                class="form-control"
                type="number"
                inputmode="numeric"
                pattern="[0-9]*"
                :placeholder="
                  currentQuestion.freeTextPlaceholder ||
                  '(nombres uniquement)'
                "
              />
              <input
                v-else
                v-model="freeTextAnswer"
                class="form-control"
                type="text"
                :placeholder="
                  currentQuestion.freeTextPlaceholder || 'Votre réponse'
                "
              />
            </div>
            <button
              @click="handleFreeTextAnswer"
              class="btn-next"
              :disabled="!freeTextAnswer"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>

          <!-- Back Button -->
          <button @click="previousQuestion" class="btn-return" v-if="canGoBack">
            Retour
          </button>
        </div>
      </div>

      <!-- Survey Complete Step -->
      <div v-else-if="isSurveyComplete" class="survey-complete">
        <h2>Merci pour votre réponse et bonne journée.</h2>
        <button @click="resetSurvey" class="btn-next">
          Nouveau questionnaire
        </button>
      </div>

      <!-- Logo -->
      <img class="logo" src="../assets/Alycelogo.webp" alt="Logo Alyce" />
    </div>

    <!-- Footer -->
    <div class="footer">
      <AdminDashboard :questions="questions" />
    </div>
    <!-- PDF Modal -->
    <div v-if="pdfToShow" class="modal">
      <div class="modal-content pdf-content">
        <span class="close" @click="pdfToShow = null">&times;</span>
        <iframe
          :src="pdfToShow"
          width="100%"
          height="500px"
          type="application/pdf"
        >
          This browser does not support PDFs. Please download the PDF to view
          it.
        </iframe>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from "vue";
import { db } from "../firebaseConfig";
import { collection, getDocs, addDoc } from "firebase/firestore";
import { doc, getDoc, setDoc } from "firebase/firestore";
import { questions } from "./surveyQuestions.js";
import CommuneSelector from "./CommuneSelector.vue";
import AdminDashboard from "./AdminDashboard.vue";
import StreetSelector from "./StreetSelector.vue";
import GareSelector from "./GareSelector.vue";

// Refs
const persistentQ1 = ref(null);
const docCount = ref(0);
const currentStep = ref("enqueteur");
const startDate = ref("");
const enqueteur = ref("");
const currentQuestionIndex = ref(0);
const answers = ref({});
const freeTextAnswer = ref("");
const questionPath = ref(["Q1"]);
const isEnqueteurSaved = ref(false);
const isSurveyComplete = ref(false);
const selectedCommune = ref("");
const stationInput = ref("");
const filteredStations = ref([]);
const pdfToShow = ref(null);
const streetSelections = ref({});
const communeSelections = ref({});
const postalCodePrefixes = ref({});
const gareSelections = ref({});
const savedPoste = ref(null);

// Firestore refs (Re-added)
const surveyCollectionRef = collection(db, "Corbeil");
const counterDocRef = doc(db, "counterCorbeil", "surveyCounter");

const firstQuestion = questions.find((q) => q.id === "Poste");

const handleGareSelection = () => {
  if (currentQuestion.value.usesGareSelector) {
    const questionId = currentQuestion.value.id;
    const selectedValue = gareSelections.value[questionId];

    if (selectedValue && selectedValue.trim() !== "") {
      answers.value[questionId] = selectedValue;
      nextQuestion();
      gareSelections.value[questionId] = "";
    }
  }
};

const handleStreetSelection = () => {
  if (currentQuestion.value.usesStreetSelector) {
    const questionId = currentQuestion.value.id;
    const selectedValue = streetSelections.value[questionId];

    if (selectedValue && selectedValue.trim() !== "") {
      answers.value[`${questionId}_RUE`] = selectedValue;
      nextQuestion();
      streetSelections.value[questionId] = "";
    }
  }
};

const currentQuestion = computed(() => {
  if (
    currentQuestionIndex.value >= 0 &&
    currentQuestionIndex.value < questions.length
  ) {
    const question = questions[currentQuestionIndex.value];

    // Handle dynamic text
    const text =
      typeof question.text === "function"
        ? question.text(answers.value)
        : question.text;

    // Handle dynamic options
    const options =
      typeof question.options === "function"
        ? question.options(answers.value)
        : question.options;

    // Debug logging for all questions
    console.log(
      `Question ${question.id} detected. Current answers:`,
      answers.value
    );
    console.log(`${question.id} text after evaluation:`, text);
    console.log(`${question.id} options after evaluation:`, options);

    return {
      ...question,
      text,
      options,
    };
  }
  return null;
});

// Methods

const currentQuestionOptions = computed(() => {
  if (
    currentQuestion.value &&
    typeof currentQuestion.value.options === "function"
  ) {
    return currentQuestion.value.options(answers.value);
  }
  return currentQuestion.value?.options || [];
});

const showFilteredStations = computed(
  () => stationInput.value.length > 0 && filteredStations.value.length > 0
);

const canGoBack = computed(() => questionPath.value.length > 1);

const isLastQuestion = computed(
  () => currentQuestionIndex.value === questions.length - 1
);

const progress = computed(() => {
  if (currentStep.value !== "survey") return 0;
  if (isSurveyComplete.value) return 100;
  const totalQuestions = questions.length;
  const currentQuestionNumber = currentQuestionIndex.value + 1;
  const isLastOrEnding =
    isLastQuestion.value ||
    currentQuestion.value?.options?.some((option) => option.next === "end");
  return isLastOrEnding
    ? 100
    : Math.min(Math.round((currentQuestionNumber / totalQuestions) * 100), 99);
});

// Methods
const setEnqueteur = () => {
  if (enqueteur.value.trim() !== "") {
    currentStep.value = "start";
    currentQuestionIndex.value = persistentQ1.value ? 1 : 0;
    isEnqueteurSaved.value = true;
  }
};

const selectPoste = (option) => {
  savedPoste.value = option.id || option.text;
  answers.value["Poste"] = option.id || option.text;
  currentStep.value = "survey";

  // Find the index of the next question based on the option's next property
  const nextQuestionIndex = questions.findIndex((q) => q.id === option.next);
  if (nextQuestionIndex !== -1) {
    currentQuestionIndex.value = nextQuestionIndex;
    questionPath.value = ["Poste", option.next];
  }
};

const startSurvey = () => {
  startDate.value = new Date().toLocaleTimeString("fr-FR", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
  currentStep.value = "poste";
  isSurveyComplete.value = false;
};

// Add this near the top of the <script setup> section
const logAnswers = () => {
  console.log("Current answers:", JSON.parse(JSON.stringify(answers.value)));
};

const selectAnswer = (option) => {
  if (currentQuestion.value && !currentQuestion.value.multipleChoice) {
    const questionId = currentQuestion.value.id;
    console.log("Selected option:", option);
    console.log("Question ID:", questionId);
    // Save option.id if it exists, otherwise save option.text
    answers.value[questionId] = option.id || option.text;
    console.log("Updated answers:", answers.value);

    if (option.next === "end") {
      finishSurvey();
    } else if (typeof option.next === "string") {
      // If next is a string, pass it to nextQuestion
      nextQuestion(option.next);
    } else if (typeof option.next === "function" && questionId === "ModeD") {
      // For ModeD options with function-based next, evaluate directly
      const nextId = option.next(answers.value);
      console.log("Function-based next ID:", nextId);
      nextQuestion(nextId);
    } else {
      // For other function-based next, let nextQuestion handle the evaluation
      nextQuestion();
    }
  }
};

const handleFreeTextAnswer = () => {
  if (currentQuestion.value) {
    answers.value[currentQuestion.value.id] = freeTextAnswer.value;
    if (currentQuestion.value.next === "end") {
      finishSurvey();
    } else {
      nextQuestion();
    }
    freeTextAnswer.value = ""; // Reset the free text answer
  }
};

const handleCommuneSelection = () => {
  if (currentQuestion.value.usesCommuneSelector) {
    const questionId = currentQuestion.value.id;
    const selectedValue = communeSelections.value[questionId];

    if (selectedValue && selectedValue.trim() !== "") {
      const parts = selectedValue.split(" - ");

      if (parts.length === 2) {
        // A commune was selected from the list
        const [commune, codeInsee] = parts;
        answers.value[`${questionId}_COMMUNE`] = commune;
        answers.value[`${questionId}_CODE_INSEE`] = codeInsee;
        answers.value[`${questionId}_COMMUNE_LIBRE`] = "";
      } else {
        // Free input was used
        answers.value[`${questionId}_COMMUNE`] = "";
        answers.value[`${questionId}_CODE_INSEE`] = "";
        answers.value[`${questionId}_COMMUNE_LIBRE`] = selectedValue.trim();
      }

      nextQuestion();
      communeSelections.value[questionId] = "";
    }
  }
};

const nextQuestion = (forcedNextId = null) => {
  let finalNextId = forcedNextId; // Use a new variable for clarity

  // If no ID was forced (e.g., coming from free text submit without specific routing), determine it now.
  // This block should generally NOT run when coming from confirmAndNextQuestion, as forcedNextId will be set.
  if (!finalNextId && currentQuestion.value) {
    console.log(`[nextQuestion] No forced ID. Determining from current question (${currentQuestion.value.id}).`);
    if (typeof currentQuestion.value.next === "function") {
       // Evaluate function-based next, typically for single-choice questions already handled in selectAnswer,
       // but could be relevant if called unexpectedly.
       finalNextId = currentQuestion.value.next(answers.value);
    } else {
       // Use the default next string from the question definition.
       finalNextId = currentQuestion.value.next;
    }
  }

  // --- Critical Logging ---
  console.log(`[nextQuestion] Attempting navigation. Received forced ID: "${forcedNextId}". Final ID to search for: "${finalNextId}"`);

  if (finalNextId === "end") {
    console.log(`[nextQuestion] Final ID is 'end'. Finishing survey.`);
    finishSurvey();
  } else if (finalNextId) {
    // --- Log before searching ---
    console.log(`[nextQuestion] Searching for index of ID: -->"${finalNextId}"<--`); // Added arrows to see spaces
    const nextIndex = questions.findIndex((q) => q.id === finalNextId);
    // --- Log the result ---
    console.log(`[nextQuestion] Result of findIndex for "${finalNextId}": ${nextIndex}`);

    if (nextIndex !== -1) {
      const nextQ = questions[nextIndex];
      console.log(`[nextQuestion] Found question: ${nextQ.id} at index ${nextIndex}. Proceeding.`);
      // Initialize answer as an array if the *next* question is multiple choice and not yet answered
      if (nextQ.multipleChoice && answers.value[nextQ.id] === undefined) {
        console.log(`[nextQuestion] Initializing answers array for upcoming multiple choice: ${nextQ.id}`);
        answers.value[nextQ.id] = [];
      }

      currentQuestionIndex.value = nextIndex;
      // Only push path if it's different from the last element to avoid duplicates on back/forth
      if (questionPath.value[questionPath.value.length - 1] !== finalNextId) {
          questionPath.value.push(finalNextId);
      }
      freeTextAnswer.value = ""; // Reset free text for the next question

      // Execute onEnter function if it exists
      if (typeof nextQ.onEnter === "function") {
        nextQ.onEnter(answers.value);
      }
    } else {
      // --- This is where the error happens ---
      console.error(`[nextQuestion] Error: Question with ID "${finalNextId}" NOT FOUND in questions array. Finishing survey.`);
      finishSurvey(); // Premature end
    }
  } else {
    console.warn(`[nextQuestion] Warning: No finalNextId could be determined. currentQuestion ID: ${currentQuestion.value?.id}. Finishing survey.`);
    finishSurvey();
  }
};

const previousQuestion = () => {
  if (canGoBack.value) {
    questionPath.value.pop();
    const previousQuestionId =
      questionPath.value[questionPath.value.length - 1];
    const previousIndex = questions.findIndex(
      (q) => q.id === previousQuestionId
    );
    if (previousIndex !== -1) {
      currentQuestionIndex.value = previousIndex;
      delete answers.value[questions[currentQuestionIndex.value].id];
      freeTextAnswer.value = "";
    }
  }
};

// New method for multiple choice selection
const toggleMultiSelect = (option) => {
  if (currentQuestion.value && currentQuestion.value.multipleChoice) {
    const questionId = currentQuestion.value.id;
    const answerArray = answers.value[questionId] || [];
    const optionValue = option.id || option.text;
    const index = answerArray.indexOf(optionValue);

    if (index === -1) {
      answerArray.push(optionValue);
    } else {
      answerArray.splice(index, 1);
    }
    answers.value[questionId] = answerArray;
    console.log("Updated multi-select answers:", answers.value);
  }
};

// New method to check if a multi-select option is selected (for styling)
const isOptionSelected = (option) => {
  if (currentQuestion.value && currentQuestion.value.multipleChoice) {
    const questionId = currentQuestion.value.id;
    const answerArray = answers.value[questionId] || [];
    const optionValue = option.id || option.text;
    return answerArray.includes(optionValue);
  }
  return false;
};

// New method to confirm multi-select and move next
const confirmAndNextQuestion = () => {
  if (currentQuestion.value && currentQuestion.value.multipleChoice) {
    console.log(`[confirmAndNextQuestion] Starting for ${currentQuestion.value.id}`);
    if (currentQuestion.value.next === "end") { // Check if the question itself points to "end"
      console.log(`[confirmAndNextQuestion] Question ${currentQuestion.value.id} next is 'end', finishing survey.`);
      finishSurvey();
    } else {
      // Determine the default next question ID
      let nextQuestionId = null;
      if (typeof currentQuestion.value.next === 'function') {
        nextQuestionId = currentQuestion.value.next(answers.value);
      } else {
        nextQuestionId = currentQuestion.value.next; // e.g., "20" for non_utilisation_velo
      }
      console.log(`[confirmAndNextQuestion] Default next ID determined as: ${nextQuestionId}`);

      // Check if ANY selected option requires precision
      const currentAnswers = answers.value[currentQuestion.value.id] || [];
      console.log(`[confirmAndNextQuestion] Current selections:`, currentAnswers);
      const precisionOptionSelected = currentQuestion.value.options.find(opt => 
        currentAnswers.includes(opt.id || opt.text) && opt.requiresPrecision
      );
      console.log(`[confirmAndNextQuestion] Precision option selected?:`, precisionOptionSelected);

      if (precisionOptionSelected && precisionOptionSelected.next) {
        // If an option requiring precision is selected, route to its specific 'next'
        console.log(`[confirmAndNextQuestion] Precision required. Routing to option's next: ${precisionOptionSelected.next}`);
        nextQuestion(precisionOptionSelected.next); // e.g., nextQuestion("non_utilisation_velo_Autre")
      } else if (nextQuestionId) {
        // Otherwise (no precision option selected), route to the default 'next' for the question
        console.log(`[confirmAndNextQuestion] No precision override. Routing to default next: ${nextQuestionId}`);
        nextQuestion(nextQuestionId); // e.g., nextQuestion("20")
      } else {
        // Fallback if no next is defined
        console.warn(`[confirmAndNextQuestion] Fallback triggered! No default next ID and no precision override for ${currentQuestion.value.id}. Calling nextQuestion() without ID.`);
        nextQuestion(); 
      }
    }
  }
};

// Update the finishSurvey function
const finishSurvey = async () => {
  isSurveyComplete.value = true;
  const now = new Date();
  logAnswers(); // Log all answers before saving to Firebase

  const uniqueId = await getNextId();

  let surveyData = {
    ID_questionnaire: uniqueId,
    HEURE_DEBUT: startDate.value || "",
    DATE: now.toLocaleDateString("fr-FR").replace(/\//g, "-"),
    JOUR: now.toLocaleDateString("fr-FR", { weekday: "long" }),
    ENQUETEUR: enqueteur.value || "",
    HEURE_FIN: now.toLocaleTimeString("fr-FR", {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
    }),
    Poste: savedPoste.value || "",
    PORT_ID_ORIGIN: answers.value.PORT_ID_ORIGIN || "",
    PORT_ID_DESTINATION: answers.value.PORT_ID_DESTINATION || "",
  };

  // Include all answers in the survey data, filtering out undefined values
  Object.keys(answers.value).forEach((key) => {
    // Only include defined values
    if (answers.value[key] !== undefined) {
      surveyData[key] = answers.value[key];
    } else {
      surveyData[key] = ""; // Use empty string for undefined values
    }
  });

  console.log("Survey data to be saved:", surveyData);

  try {
    await addDoc(surveyCollectionRef, surveyData);
    console.log("Survey data saved successfully");
  } catch (error) {
    console.error("Error saving survey data:", error);
  }
};

// Update the resetSurvey function
const resetSurvey = () => {
  currentStep.value = "start";
  startDate.value = "";
  answers.value = { Q1: persistentQ1.value };
  currentQuestionIndex.value = 1; // Start from Q2
  isSurveyComplete.value = false;
};

const getDocCount = async () => {
  try {
    const querySnapshot = await getDocs(surveyCollectionRef);
    docCount.value = querySnapshot.size;
  } catch (error) {
    console.error("Error getting document count:", error);
  }
};

const getNextId = async () => {
  const counterDoc = await getDoc(counterDocRef);
  let counter = 1;

  if (counterDoc.exists()) {
    counter = counterDoc.data().value + 1;
  }

  await setDoc(counterDocRef, { value: counter });

  return `Corbeil-${counter.toString().padStart(6, "0")}`;
};
</script>


<style>
/* Base styles */
body {
  background-color: #2a3b63;
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}

.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background-color: #2a3b63;
  color: white;
}

/* Center the Start Survey button horizontally and vertically */
.start-survey-container {
  justify-content: center;
  /* Center horizontally */
  align-items: center;
  /* Center vertically */
  height: 50vh;
  /* Full viewport height */
  width: 100%;
  /* Full width */
  margin-bottom: 5%;
}

.content-container {
  flex-grow: 1;
  /* This allows the content to take up available space */
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5% 0;
  width: 90%;
  max-width: 600px;
  margin: 0 auto;
  box-sizing: border-box;
  overflow-y: auto;
  /* Allow scrolling if content overflows */
}

.question-container {
  width: 100%;
  margin-bottom: 30px;
}

.input-container {
  display: flex;
  justify-content: center;
  /* Center horizontally */
  width: 100%;
  /* Take full width of the parent */
}

h2 {
  text-align: center;
  width: 100%;
}

.form-control {
  width: 100%;
  max-width: 400px;
  /* Maximum width of the input */
  padding: 10px;
  border-radius: 5px;
  border: 1px solid white;
  background-color: #333;
  color: white;
  font-size: 16px;
  margin-bottom: 15px;
  box-sizing: border-box;
  outline: none;
}

.btn-next,
.btn-return,
.btn-option {
  width: 100%;
  max-width: 400px;
  color: white;
  padding: 15px;
  margin-top: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

.btn-next {
  background-color: green;
}

.btn-return {
  background-color: grey;
  margin-top: 30px;
}

.btn-option {
  background-color: #4a5a83;
  text-align: left;
}

.logo {
  max-width: 25%;
  height: auto;
  margin-top: 40px;
  margin-bottom: 20px;
}

.footer {
  background: linear-gradient(to right, #4c4faf, #3f51b5);
  padding: 20px;
  text-align: center;
  width: 100%;
  box-sizing: border-box;
  position: relative;
  /* Keep the footer relative to its parent */
}

.btn-download {
  background-color: #ffffff;
  color: #4c4faf;
  border: none;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  padding: 10px 20px;
  border-radius: 25px;
  transition: all 0.3s ease;
  margin-bottom: 15px;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.doc-count {
  font-size: 14px;
  opacity: 0.9;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background-color: #e0e0e0;
  position: relative;
  overflow: hidden;
  margin-bottom: 20px;
}

.progress {
  height: 100%;
  background-color: #4caf50;
  transition: width 0.3s ease-in-out;
}

.commune-dropdown {
  max-height: 200px;
  overflow-y: auto;
  border: 1px solid #ccc;
}

.commune-option {
  padding: 5px;
  cursor: pointer;
}

.commune-option:hover {
  background-color: #f0f0f0;
}

@media screen and (max-width: 768px) {
  .question-container {
    margin-bottom: 20px;
  }

  .btn-return {
    margin-top: 20px;
  }

  .logo {
    margin-top: 30px;
  }
}

/* Ensure responsive centering */
@media screen and (max-width: 480px) {
  .form-control {
    max-width: 100%;
    /* Ensure full width on small screens */
  }
}
.btn-pdf {
  background-color: #ff9800;
  /* Orange color to make it distinct */
  color: white;
  padding: 15px;
  margin: 10px 0;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  width: 100%;
  max-width: 400px;
  text-align: center;
  transition: background-color 0.3s;
}

.btn-pdf:hover {
  background-color: #f57c00;
  /* Darker orange on hover */
}

.modal {
  display: flex;
  position: fixed;
  z-index: 1;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: auto;
  background-color: rgba(0, 0, 0, 0.4);
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: #fefefe;
  padding: 20px;
  border: 1px solid #888;
  width: 90%;
  max-width: 800px;
  position: relative;
}

.pdf-content {
  height: 80vh;
  display: flex;
  flex-direction: column;
}

.close {
  color: #aaa;
  float: right;
  font-size: 28px;
  font-weight: bold;
  cursor: pointer;
  position: absolute;
  right: 10px;
  top: 5px;
}

.close:hover,
.close:focus {
  color: black;
  text-decoration: none;
  cursor: pointer;
}

/* Ensure the PDF fits within the modal */
.pdf-content iframe {
  flex-grow: 1;
  border: none;
  margin-top: 20px;
}

/* Add these new styles */
.precision-input {
  margin-top: 15px;
}

.precision-input h3 {
  font-size: 1.1em;
  margin-bottom: 10px;
}

.btn-option.selected {
  background-color: #6a8acd;
  border: 2px solid white;
}
</style>