<template>
  <div>
    <div v-if="state == States.MODESELECT">
      <h2>Выберите режим</h2>
      <div class="options">
        <div :class="{ option: true, selected: mode === key}" v-for="(obj, key) in Modes" :key="key" @click="selectMode(key)">{{ obj.desc }}</div>
      </div>
      <div :class="{ button: true, disabled: mode === null }" @click="startGame">Продолжить</div>
      <div class="container">
        <h3>Состояние базы</h3>
        Число книг: {{ dbState.total }}<br>
        Число поэм: {{ dbState.verses }}<br>
        Число пьес: {{ dbState.plays }}<br>
        Коэффициент сходства: {{ dbState.k }}
      </div>
    </div>
    
    <div v-if="state === States.GAME || state === States.SELECTED">
      <div class="status">
        {{ questionIndex + 1 }} / {{ Modes[this.mode].questionsCount }}
      </div>
      <div class="container starting">
        <p v-for="p in questions[questionIndex].question.starting.split('\n')" :key="p.slice(0, 100)">
          {{ p }}
        </p>
        <!-- {{ questions[questionIndex].question.starting }} -->
      </div>
      <div v-if="questions[questionIndex].options.length > 0">
        <div class="options">
          <div :class="{ option: true, selected: optionId === option.id, red: redOptionId === option.id, disabled: state === States.SELECTED}" v-for="option in questions[questionIndex].options" :key="option.id + option.title" @click="selectOption(option)">
            <b>{{ option.title }}</b><br>{{ option.author }}
          </div>
        </div>
        <div :class="{ button: true, disabled: optionId == null }" @click="checkOptionAnswer" v-if="state === States.GAME">
          Ответить
        </div>
        <div :class="{ button: true }" @click="nextQuestion" v-if="state === States.SELECTED">
          Следующий вопрос
        </div>
      </div>
      <div v-else>
        <div class="container">
          <label>Название книги: </label>
          <input type="text" v-model="userInputTitle" :class="{ 'green': state === States.SELECTED && comapareStrings(userInputTitle, questions[questionIndex].question.title),
                                                                'red': state === States.SELECTED && !comapareStrings(userInputTitle, questions[questionIndex].question.title)
            }"/><br>
          <label>Автор: </label>
          <input type="text" v-model="userInputAuthor" :class="{ 'green': state === States.SELECTED && comapareStrings(userInputAuthor, questions[questionIndex].question.author),
                                                                 'red': state === States.SELECTED && !comapareStrings(userInputAuthor, questions[questionIndex].question.author) 
          }"/>
        </div>
        <div :class="{ button: true, disabled: !(userInputTitle || userInputAuthor) }" @click="checkTextAnswer" v-if="state === States.GAME">
          Ответить
        </div>
        <div :class="{ button: true }" @click="nextQuestion" v-if="state === States.SELECTED">
          Следующий вопрос
        </div>
      </div>

      <div class="good-message">
        {{ message }}
      </div>
      <div class="error-message">
        {{ errorMessage }}
      </div>
      
    </div>

    <div v-if="state === States.RESULTS">
      <div class="container">
      Вы ответили верно на {{ results.filter(x => x.result > 1.9).length }} вопросов из {{ Modes[mode].questionsCount }}.
      <table>
        <tr><td>Вопрос</td><td>Ваш ответ</td><td>Верный ответ</td><td>Верно?</td></tr>
        <tr v-for="result in results" :key="result.answer.id">
          <td>{{ result.answer.starting.slice(0, 100) + '...' }}</td>
          <td><b>{{ result.userAnswer.title }}</b><br>{{ result.userAnswer.author }}</td>
          <td><b>{{ result.answer.title }}</b><br>{{ result.answer.author }}</td>
          <td>{{ result.result > 1.9 ? '✔️' : ( result.result > 0.9 ? '✔️❌' : '❌' ) }}</td>
        </tr>
      </table>
      </div>
      <div :class="{ button: true, disabled: mode === null }" @click="restartGame">Сыграть еще раз</div>
    </div>
  </div>
</template>

<script>
import _ from 'underscore';
import stringSimilarity from 'string-similarity';

import BookData from '@/assets/final_data.json';

const STRING_SIMILARITY_K = 0.5;

export default {
  name: 'MainView',
  data: function () {
    return {
      mode: null,
      state: null,
      questions: null,
      questionIndex: 0,
      optionId: null,
      redOptionId: null,
      results: [],
      message: '',
      errorMessage: '',

      userInputAuthor: '',
      userInputTitle: '',

      dbState: {},
      
      // Consts
      States: {
        MODESELECT: 1,
        GAME: 2,
        SELECTED: 4,
        RESULTS: 3
      },
      Modes: {
        SELECT4: {
          desc: 'Выбор из 4 вариантов',
          optionsCount: 4,
          questionsCount: 10
        },
        GUESS: {
          desc: 'Без вариантов ответа',
          optionsCount: 0,
          questionsCount: 10
        }
      }
    }
  },
  mounted: function () {
    this.state = this.States.MODESELECT;

    this.dbState.total = BookData.length;
    this.dbState.verses = BookData.filter(x => x.type === 'v').length;
    this.dbState.plays = BookData.filter(x => x.type === 'p').length;
    this.dbState.k = STRING_SIMILARITY_K;
  },
  methods: {
    comapareStrings(first, second) {
      if (second.includes(first) && first.length > 7) {
        return true;
      } else if (first.includes(second) && second.length > 7) {
        return true;
      }
      return stringSimilarity.compareTwoStrings(first, second) > STRING_SIMILARITY_K;
    },
    selectMode: function (mode) {
      this.mode = mode;
    },
    selectOption: function (option) {
      this.optionId = option.id;
    },
    startGame: function () {
      let questionsCount = this.Modes[this.mode].questionsCount;
      let optionsCount = this.Modes[this.mode].optionsCount;

      let questions = _.sample(BookData, questionsCount);
      
      this.questions = [];
      
      for (let i = 0; i < questionsCount; i++) {
        this.questions.push({
          question: questions[i],
          options: optionsCount === 0 ? [] : _.shuffle((_.sample(BookData.filter(b => b.id != questions[i].id && b.type === questions[i].type), optionsCount - 1)).concat(questions[i]))
        })
      }
      this.state = this.States.GAME;
    },
    checkOptionAnswer: function () {
      this.results.push({
        answer: this.questions[this.questionIndex].question,
        userAnswer: this.questions[this.questionIndex].options.filter(x => x.id === this.optionId)[0],
        result: this.questions[this.questionIndex].question.id === this.optionId ? 2 : 0
      });

      if (this.questions[this.questionIndex].question.id === this.optionId) {
        this.message = 'Верно!';
      } else {
        this.errorMessage = 'Неверно!';
        this.redOptionId = this.optionId;
        this.optionId = this.questions[this.questionIndex].question.id;
      }

      this.state = this.States.SELECTED;
    },
    checkTextAnswer: function () {
      let q = this.questions[this.questionIndex].question;
      let resultAuthor = this.comapareStrings(this.userInputAuthor, q.author);
      let resultTitle = this.comapareStrings(this.userInputTitle, q.title);
      
      this.results.push({
        answer: this.questions[this.questionIndex].question,
        userAnswer: {
          title: this.userInputTitle,
          author: this.userInputAuthor,
        },
        result: resultTitle + resultAuthor
      });

      if (resultAuthor > STRING_SIMILARITY_K && resultTitle > STRING_SIMILARITY_K) {
        this.message = `Верно! Ответ: ${q.title} ${q.author}`;
      } else if (resultAuthor > STRING_SIMILARITY_K) {
        this.errorMessage = `Вы отгадали только автора! Ответ: ${q.title} ${q.author}`;
      } else if (resultTitle > STRING_SIMILARITY_K) {
        this.errorMessage = `Вы отгадали только назавние книги! Ответ: ${q.title} ${q.author}`;
      } else {
        this.errorMessage = `Ваш ответ неверен! Ответ: ${q.title} ${q.author}`;
      }

      this.state = this.States.SELECTED;
    },
    nextQuestion: function () {
      this.message = this.errorMessage = '';
      this.optionId = this.redOptionId = null;
      this.userInputAuthor = this.userInputTitle = '';

      this.state = this.States.GAME;
      this.questionIndex++;

      if (this.questionIndex === this.Modes[this.mode].questionsCount) {
        console.log(this.results);
        this.state = this.States.RESULTS;
      }
      
    },
    restartGame: function () {
      this.message = this.errorMessage = '';
      this.optionId = this.redOptionId = null;
      this.mode = null;
      this.state = this.States.MODESELECT;
      this.questions = null;
      this.questionIndex = 0;
      this.results = [];
      this.userInputAuthor = this.userInputTitle = '';
    }
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Roboto&family=Roboto+Slab&display=swap');



.options {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: space-around;
  max-width: 800px;
  margin: auto;
  border: 1px solid black;
  margin-bottom: 10px;
}

.option {
  padding: 10px;
  margin: 5px;
  border: 1px solid black;
  cursor: pointer;
}

.option.selected {
  border: 3px solid green;
  padding: 8px;
}

.option.red {
  border: 3px solid red;
  padding: 8px;
}

.disabled {
  pointer-events: none;
  background-color: #cccccc;
  color: #666666;
}

.button {
  padding: 10px;
  cursor: pointer;
  border: 1px solid black;
  max-width: 150px;
  margin: auto;
  margin-bottom: 10px;
}

.container {
  max-width: 600px;
  text-align: left;
  margin: auto;
  border: 1px solid black;
  padding: 10px;
  margin-bottom: 10px;
}

.starting {
  font-family: 'Roboto Slab', serif;
}

.results table {
  margin: auto;
  border-collapse: collapse;
}

.results table, th, td {
  border: 1px solid black;
}

.results td {
  padding: 5px;
}

input {
  width: 100%;
  margin-right: 15px;
}

input.red {
  border: 1px solid red;
}

input.green {
  border: 1px solid green;
}

</style>
