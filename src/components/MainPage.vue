<template>
  <div class="hello">
    <h1>{{ languageSet[language].header }}</h1>
    <div class="input">
      <label for="language">{{ languageSet[language].choose }}</label>
      <select v-model="language" id="language">
        <option v-for="(value, key) in languageSet" :value="key" :key="key">{{ value.name }}</option>
      </select>
    </div>
    <h3>{{ languageSet[language]['input-for-prediction'] }}</h3>
    <div class="input">
      <label for="weight">{{ languageSet[language].weight }}</label>
      <input type="number" id="weight" v-model="weight"/>
    </div>
    <div class="input">
      <label for="height">{{ languageSet[language].height }}</label>
      <input type="number" id="height" v-model="height"/>
    </div>
    <div class="input">
      <label for="gender">{{ languageSet[language].gender }}</label>
      <select id="gender" v-model="gender">
        <option value="Male">{{ languageSet[language].male }}</option>
        <option value="Female">{{ languageSet[language].female }}</option>
      </select>
    </div>
    <button @click="predict" class="button">{{ languageSet[language].predict }}</button>
    <br>
    <button class="show-table" type="button" @click="showSizeTable">{{ languageSet[language]['choose-size-yourself'] }}</button>
    <div v-show="showSizeTableFlag">
      <img alt="Size table" src="../assets/table.png">
    </div>
    <br>
<!--    <h3>{{ languageSet[language]['check-accuracy'] }}</h3>-->
<!--    <div v-if="total">-->
<!--      <p>{{ languageSet[language].total }}: {{ total }}</p>-->
<!--      <p>{{ languageSet[language].correct }}: {{ correct }}</p>-->
<!--      <p>{{ languageSet[language].accuracy }}: {{ total ? (correct / total * 100).toFixed(2) : 0 }}%</p>-->
<!--    </div>-->
<!--    <button v-if="!continueTesting" @click="startTesting" class="button">{{ languageSet[language]['start-test'] }}</button>-->
<!--    <button v-if="continueTesting" @click="stopTesting" class="button">{{ languageSet[language]['stop-test'] }}</button>-->
  </div>
</template>

<script>

export default {
  name: 'MainPage',
  data() {
    return {
      height: 0,
      weight: 0,
      gender: 'Male',
      size: 0,
      correct: 0,
      showSizeTableFlag: false,
      total: 0,
      maleClassifier: null,
      femaleClassifier: null,
      testData: [],
      testingCaseNo: 0,
      continueTesting: false,
      trained: false,
      language: 'vi',
      languageSet: {
        'en' : {
          'name' : 'English',
          'choose' : 'Choose language',
          'header' : 'Clothes size predictor App',
          'input-for-prediction' : 'Input your information and we will suggest you the best size',
          'choose-size-yourself' : 'Click here to choose size yourself from the table',
          'weight' : 'Weight (kg):',
          'height' : 'Height (cm):',
          'gender' : 'Gender:',
          'male' : 'Male',
          'female' : 'Female',
          'predict' : 'Predict',
          'start-test' : 'Start',
          'stop-test' : 'Stop',
          'check-accuracy' : 'Click below to check the accuracy of the model',
          'total' : 'Total',
          'correct' : 'Correct',
          'accuracy' : 'Accuracy',
          'training' : 'Please wait when we are initializing the model...',
          'answer' : 'Our prediction for your clothes size is: '
        },
        'vi' : {
          'name' : 'Ti???ng Vi???t',
          'choose' : 'Ch???n ng??n ng???',
          'header' : '???ng d???ng D??? ??o??n k??ch c??? qu???n ??o',
          'input-for-prediction' : 'Nh???p th??ng tin c???a b???n v?? ch??ng t??i s??? ????a ra k???t qu??? d??? ??o??n.',
          'choose-size-yourself' : 'Nh???n v??o ????y ????? t??? ch???n k??ch c??? qu???n ??o c???a m??nh b???ng b???ng d?????i ????y',
          'weight' : 'C??n n???ng (kg):',
          'height' : 'Chi???u cao (cm):',
          'gender' : 'Gi???i t??nh:',
          'male' : 'Nam',
          'female' : 'N???',
          'predict' : 'D??? ??o??n',
          'start-test' : 'B???t ?????u',
          'stop-test' : 'D???ng',
          'check-accuracy' : 'Nh???n v??o button ph??a d?????i ????? ki???m tra ????? ch??nh x??c c???a m?? h??nh',
          'total' : 'T???ng s???',
          'correct' : 'S??? case ????ng',
          'accuracy' : '????? ch??nh x??c',
          'training' : 'Vui l??ng ?????i trong khi ch??ng t??i kh???i t???o m?? h??nh...',
          'answer' : 'K??ch c??? qu???n ??o c???a b???n l??: '
        }
      }
    }
  },
  async created() {
    this.trained = false;
    await this.train();
    this.trained = true;
    this.testPrepare();
  },
  methods: {
    train: async function () {
      if (this.trained) {
        return;
      }
      let bayes = require('node-bayes');
      const response = await fetch('/vn_train.csv');
      const data = await response.blob();
      const file = new File([data], name, {
        type: data.type || "text/plain",
      });
      let columnArr = [];
      let dataArrForMale = [];
      let dataArrForFemale = [];
      const reader = new FileReader();
      reader.readAsText(file);
      reader.onload = () => {
        const lines = reader.result.split('\n');
        lines.forEach(line => {
          const [gender, weight, height, size] = line.split(',');
          if (gender === 'Gender') {
            columnArr = ['Weight', 'Height', 'Size'];
          } else if (weight && gender && height) {
            if (gender === 'Male') {
              dataArrForMale.push([parseInt(weight), parseInt(height), size]);
            } else {
              dataArrForFemale.push([parseInt(weight), parseInt(height), size]);
            }
          }
        });

        this.maleClassifier = new bayes.NaiveBayes({
          columns: columnArr,
          data: dataArrForMale,
          verbose: true
        });
        this.maleClassifier.train();

        this.femaleClassifier = new bayes.NaiveBayes({
          columns: columnArr,
          data: dataArrForFemale,
          verbose: true
        });
        this.maleClassifier.train();
        this.femaleClassifier.train();
        this.trained = true;
      }
    },
    predict: function() {
      if (!this.trained) {
        alert(this.languageSet[this.language].training);
        return;
      }
      let answer
      if (this.gender === 'Male') {
        answer = this.maleClassifier.predict([this.weight, this.height]);
      } else {
        answer = this.femaleClassifier.predict([this.weight, this.height]);
      }
      alert(this.languageSet[this.language].answer + answer.answer);
      console.log([this.weight, this.height], answer);
    },
    testPrepare: async function() {
      if (!this.trained) {
        alert(this.languageSet[this.language].training);
        return;
      }
      this.correct = 0;
      this.total = 0;
      const response = await fetch('/test.csv');
      const data = await response.blob();
      const file = new File([data], name, {
        type: data.type || "text/plain",
      });
      const reader = new FileReader();
      reader.readAsText(file);
      reader.onload = () => {
        const lines = reader.result.split('\n');
        lines.forEach(line => {
          const [gender, weight, height, size] = line.split(',');
          if (gender && weight && height && size)
            this.testData.push([gender, weight, height, size])
        });
      }
    },
    startTesting: function() {
      if (!this.trained) {
        alert(this.languageSet[this.language].training);
        return;
      }
      this.continueTesting = true;
      this.testSingleCase()
    },
    stopTesting: function() {
      this.continueTesting = false;
    },
    showSizeTable: function() {
      this.showSizeTableFlag = !this.showSizeTableFlag;
    },
    testSingleCase() {
      let [gender, weight, height, size] = this.testData[this.testingCaseNo];

      let answer
      if (gender === 'Male') {
        answer = this.maleClassifier.predict([weight, height]);
        if (answer.answer === size) {
          this.correct++;
        } else {
          console.log('Wrong answer: ', [gender, weight, height, size], answer.answer);
        }
        this.total++;
      } else if (gender === 'Female') {
        answer = this.femaleClassifier.predict([weight, height]);
        if (answer.answer === size) {
          this.correct++;
        } else {
          console.log('Wrong answer: ', [gender, weight, height, size], answer.answer);
        }
        this.total++;
      }

      if (this.continueTesting) {
        this.testingCaseNo++;
        setTimeout(() => {
          this.testSingleCase();
        }, 0);
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}

button {
  background-color: #42b983;
  border: none;
  color: white;
  padding: 15px 32px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 16px;
  margin: 4px 2px;
  cursor: pointer;
}

.input {
  margin: 20px 0;
}
.input label {
  display: inline-block;
  width: 200px;
  text-align: left;
}
.input input {
  width: 200px;
  padding: 8px;
}
.input select {
  width: 220px;
  padding: 8px;
}
.show-table {
  background-color: grey;
}
</style>
