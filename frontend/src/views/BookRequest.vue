<template>
  <div class="container-body">
    <!--loading-->
    <div class="spinner-border text-primary" role="status" v-if="loading === true">
    </div>
    <div v-else>
      <h1>진료 신청서</h1>
    <div class="container-card">
        <div class="card">
          <div class="card-body">
            <!-- <img src="../assets/images/doctor.jpg" class="img-thumbnail" alt="doctorImg" id="doctorImg"/> -->
            <!-- <p class="card-title">{{this.getDoctorName}}</p> -->
            <div id="date-select">
              <div class="container-doctorName">
                <p class="card-title">{{this.getDoctorName}}</p>
              </div>
              <div class="container-date-picker">
                <label class="date-picker-label">예약 날짜</label>
                <input type='date' class="form-control" @change="datePicked()" v-model="date"/>
              </div>
            </div>
            <div class="time-container" v-if="timeList.length !== 0">
              <div v-for="(time, idx) in allTime" :key="idx">
                <input type="radio" :id="`avail-time-${idx}`" name="avail-time"/>
                <label :class="`btn btn-outline-primary avail-time-label ${timeCheckList[idx] ? '' : 'time-disabled'}`" :for="`avail-time-${idx}`" @click="appointmentTime = time;">{{time}}</label>
              </div>
            </div>
            <div class="no-time-avail" v-else>
            휴무일이거나 예약 가능한 시간이 없습니다.
            </div>
            <div class="symptom-container">
              <div class="container-upload">
                <p style="text-align: left; font-weight: bold; font-size: 20px;">증상 사진 첨부</p>
                <input @change="upload" multiple type="file" accept="image/*">
              </div>
              <div class="form-floating">
                <textarea placeholder="증상에 대해 자세히 입력해주세요." id="symptom-textarea" v-model="symptom"></textarea>
              </div>
            </div>
            <div class="d-grid gap-2 btn-container">
              <a href="#" class="btn btn-primary" @click="bookRequest()">진료 예약하기</a>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>  
</template>

<script>

import { mapGetters, mapMutations, mapState } from 'vuex';
import axios from 'axios';
export default {
  data() {
    return {
      date: '',
      allTime: [],
      timeList: [],
      timeCheckList:[],
      symptom: '',
      doctorName: '',
      doctorId: '',
      departmentName: '',
      appointmentTime: undefined,
      apptDateTime: '',
      loading : true,
    }
  },
  computed: {
    ...mapGetters(['getDoctorIdKept', 'getDoctorName', 'getDepartmentName']),
    ...mapState(['doctorDetail', 'photoUrl']),
  },
  methods: {
    upload(e){
      const formData = new FormData();
      var files = e.target.files;
      const maxSize = 10* 1024* 1024;//최대 용량 10MB
      var fileSize = 0;
      var type = '';
      //파일 타입체크 및 용량 계산
      for (let i = 0; i < files.length; i++) {
        type = files[i].type.toLowerCase();
        fileSize += files[i].size;
        if(type === 'image/png' || type === 'image/jpg' || type === 'image/jpeg' || type === 'image/gif'){
          formData.append("images", files[i]);
        }else{
          alert('이미지 첨부는 jpg, jpeg, png, gif 파일만 가능합니다!');
          e.target.value = '';//첨부파일이 없습니다 세팅
          return;   
        }
      }
      //용량 체크
      if(fileSize > maxSize){
        alert('파일 첨부는 10MB 까지만 가능합니다!');
        return;
      }
      if(e.target.value !== ''){
        this.$store.dispatch('upload', formData).then((res) => {
        console.log(res.data);
        this.setphotoUrl(res.data);
        });
      }
    },
    ...mapMutations(['setBookInfo', 'setDoctorDetail', 'setDoctorIdKept', 'setDoctorName', 'setDepartmentName', 'setphotoUrl']),
    async datePicked() {
      const response = await axios.get(`/api/doctor/opening-hours/${this.getDoctorIdKept}/${this.date}`);
      this.allTime = response.data.result;

      this.timeList = [];
      this.timeCheckList = [];
      //날짜 선택하면 기능
      const param = {
        doctorId: this.getDoctorIdKept,
        selectedDate: this.date,
      };
      console.log(param);
      this.$store.dispatch('setAvailTime', param).then((res) => {
        console.log(res.data.result);
        //받아온 timeList 시간 추출
        let hour = '';
        let minutes = '';
        res.data.result.forEach(element => {
          hour = new Date(element).getHours();
          minutes = new Date(element).getMinutes();
          if (hour < 10) {
            hour = '0' + hour;
          }
          if (minutes === 0) {
            minutes = '00';
          }
          element = hour + ':' + minutes;
          this.timeList.push(element);
        });
        console.log(this.timeList);

        //예약이 있는 시간 리스트 추출, 예약 가능시 true, 불가시 false
        for (let i = 0; i < this.allTime.length; i++) {
          if (this.timeList.includes(this.allTime[i])) {
            this.timeCheckList.push(true);
          } else {
            this.timeCheckList.push(false);
          }
        }

        console.log([...this.timeCheckList]);

        //선택한 예약시간이 없을 경우(날짜 선택만 된 초기) 기본 default값 가장 첫번째 예약 가능시간으로 설정
        if (this.appointmentTime === '') {
          this.appointmentTime = res.data.result[0];
        }
        console.log(this.appointmentTime);
      });
    },
    bookRequest() {
      //validation 추가, date, symptom 없을 경우 못하게
      console.log(this.appointmentTime);
      if (this.symptom === '') {
        alert('증상을 입력해주세요!');
        return;
      } else if (!this.appointmentTime) {
        alert('예약 날짜와 시간을 선택해주세요!');
        return;
      }

      if (!confirm('예약을 확정하시겠습니까?')) {
        return;
      }

      if (this.appointmentTime.length === 4) {
        this.apptDateTime = this.date + ' 0' + this.appointmentTime;
      } else {
        this.apptDateTime = this.date + ' ' + this.appointmentTime;
      }
      let bookReqInfo = {
        "patientId": localStorage.getItem('userId'),
        "doctorId": this.getDoctorIdKept,
        "appointmentTime": this.apptDateTime,
        "symptom": this.symptom,
        "departmentName": this.getDepartmentName,
        "charge": this.doctorDetail.charge,
        "symptomImages": this.photoUrl,
      };
      console.log(bookReqInfo);
      this.$store.dispatch('setBookReq', bookReqInfo).then((res) => {
        console.log(res.data);
        this.$router.push({
          name: 'bookConfirm',
            params: {
            patientName: res.data.patientName,
            doctorName: res.data.doctorName,
            departmentName: res.data.departmentName,
            charge: res.data.charge,
            appointmentTime: res.data.appointmentTime,
          }
        });
      }).catch(error => {
        console.log(error)
        alert('해당 날짜에 예약이 불가합니다!');
      });
    }
  },
  created() {
    //파라미터 값들 저장
    if(this.$route.params.doctorId !== '' && this.$route.params.doctorId !== undefined && this.$route.params.doctorId !== null){
      this.setDoctorIdKept(this.$route.params.doctorId);
    }
    if(this.$route.params.doctorName !== '' && this.$route.params.doctorName !== undefined && this.$route.params.doctorName !== null){
      this.setDoctorName(this.$route.params.doctorName);
    }
    if(this.$route.params.departmentName !== '' && this.$route.params.departmentName !== undefined && this.$route.params.departmentName !== null){
      this.setDepartmentName(this.$route.params.departmentName);
    }
    //의사 상세정보 api
    this.$store.dispatch('getDoctorDetail', this.getDoctorIdKept).then((res) => {
      console.log(res.data);
      this.setDoctorDetail(res.data);
      this.loading = false;
    });
    //오늘 날짜 설정
    let todayDate = new Date();
    let yy = String(todayDate.getFullYear());
    let month = todayDate.getMonth() + 1;
    let mm = month < 10 ? '0' + month : month;
    let dd = String(todayDate.getDate() < 10 ? '0' + todayDate.getDate() : todayDate.getDate());

    //오늘 date 값 입력
    let date = yy + '-' + mm + '-' + dd;
    this.date = date;

    this.datePicked();
  },
}
</script>

<style>
#doctorImg{
  width:35%;
  height:35%;
  margin: 10px;
}

#times{
  display: inline;
  margin: 5px;
}

#badge{
  font-size: 15px;
  padding:10px 20px;
  margin: 10px 0px;
}

#badge:hover{
  cursor:pointer;
}

.btn-container {
  margin-top: 10px;
}

#date-select{
  display: flex;
  justify-content: space-between;
  /* align-items: center; */
}

.time-container {
  margin-top: 20px;
	display: grid;
  gap: 10px;
	grid-template-columns: repeat(4, 1fr);
}

.time-container input {
  display: none;
}

.avail-time-label {
  width: 100%;
}

.time-container input:checked+label {
  color: #fff;
  background-color: #0d6efd;
}

.time-disabled{
  pointer-events: none;
  background-color: lightgray;
  color:rgb(148, 146, 146);
  border:1px solid rgb(148, 146, 146);
}

.symptom-container{
  margin-top: 20px;
  text-align: left;
}

#symptom-textarea{
  width: 100%;
  resize: none;
  border: 1px solid lightgray;
  padding: 10px;
  outline: none;
  border-radius: 5px;
  height: 100px;
}

.no-time-avail{
  margin-top: 20px;
  font-weight: bold;
  font-size: 20px;
}

.date-picker-label {
  margin-right: 10px;
  width:150px;
  line-height: 45px;
}

.container-date-picker{
  display: flex;
}

.container-doctorName{
  display: flex; 
  flex-direction: column; 
  justify-content: center;
}

.container-upload{
  margin-bottom: 20px;
}
</style>