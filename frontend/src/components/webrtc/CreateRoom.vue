<template>
<div>
  <el-button type="primary" icon="el-icon-folder-add" @click="dialogFormVisible = true" circle></el-button>
  <el-dialog title="화상채팅방 생성" v-model="dialogFormVisible">
    <el-form :model="form" >
    <el-form-item prop="name" label="방 이름" :label-width="formLabelWidth"
      :rules="[
        { required: true, message: '필수 입력 항목입니다.', trigger: 'blur' },
        {
          pattern: /^[ㄱ-ㅎ가-힣a-zA-Z0-9_-]+$/g,
          message: '한글, 영어, 숫자, -, _ 이외에는 사용할 수 없습니다.',
          trigger: 'change'
        }
      ]"
    >
      <el-input v-model="form.name" autocomplete="off"></el-input>
    </el-form-item>
    <el-form-item prop="keyword" label="키워드" id="kTag" :label-width="formLabelWidth">
        <el-tag
        :key="tag"
        v-for="tag in dynamicTags"
        closable
        :disable-transitions="false"
        @close="handleClose(tag)"
        id="kTag">
        {{tag}}
        </el-tag>
        <el-input
        class="input-new-tag"
        v-if="inputVisible"
        v-model="inputValue"
        ref="saveTagInput"
        size="mini"
        @keyup.enter="handleInputConfirm"
        @blur="handleInputConfirm"
        >
        </el-input>
        <el-button v-else class="button-new-tag" size="small" @click="showInput">+ New Tag</el-button>
    </el-form-item>
    <el-form-item prop="personNum" label="인원" :label-width="formLabelWidth">
      <el-input-number v-model="num" id = "pNum" controls-position="right" @change="handleChange" :min="1" :max="10"></el-input-number>
    </el-form-item>
    <el-form-item prop="roomTheme" label="카테고리" :label-width="formLabelWidth">
      <el-select v-if="curPage" v-model="bValue" placeholder="Book">
        <el-option
          v-for="item in b_options"
          :key="item.bookCategoryId"
          :label="item.bookCategory"
          :value="item.bookCategoryId">
        </el-option>
      </el-select>
      <el-select v-else v-model="mValue" placeholder="Movie">
        <el-option
          v-for="item in m_options"
          :key="item.movieCategoryId"
          :label="item.movieCategory"
          :value="item.movieCategoryId">
        </el-option>
      </el-select>
    </el-form-item>
    <el-form-item prop="roomLock" label="방 잠금" :label-width="formLabelWidth">
      <el-checkbox v-model="checkedLock" @change="handleCheckbox"></el-checkbox>
      <el-input v-model="form.pwd" v-show="isLocked" placeholder="비밀번호를 입력하세요." onfocus="this.placeholder=''" onblur="this.placeholder='비밀번호를 입력하세요.'"></el-input>
    </el-form-item>
      <div class="container">
         <div class="wrapper">
            <div class="image">
               <img id="thumbnail-image" src=" " alt=" ">
            </div>
            <div class="content">
               <div class="icon">
                  <i class="fas fa-cloud-upload-alt"></i>
               </div>
               <div class="text">
                  방생성 썸네일 이미지를 추가해주세요.
               </div>
            </div>
            <div id="cancel-btn" @click="removeImgAction">
               <i class="fas fa-times"></i>
            </div>
            <div class="file-name">
               {{ fileName }}
            </div>
         </div>
         <button @click="chooseFile" type="button" id="custom-btn">Choose a file</button>
         <input id="file-input" type="file" hidden @change="getFile">
      </div>
    </el-form>
    <template #footer>
    <span class="dialog-footer">
      <el-button type="primary" @click="formRoom()" >생성</el-button>
      <el-button @click="formClose()">취소</el-button>
    </span>
    </template>
  </el-dialog>
  <el-dialog v-model="dialogVisible">
    <img width="100%" :src="dialogImageUrl" alt="" />
  </el-dialog>
</div>
</template>
<script>
import axios from 'axios';
import { OpenVidu } from 'openvidu-browser';
import { useStore } from 'vuex';
import { useRouter } from "vue-router"
import { wrap } from 'lodash';

let regExp = /[0-9a-zA-Z\^\&\'\@\{\}\[\]\,\$\=\!\-\#\(\)\.\%\+\~\_ ]+$/;

export default {
  name: 'CreateRoom',
  data() {
    const store = useStore();
    return {
      dynamicTags: ['키워드를 입력하세요'],
      inputVisible: false,
      inputValue: '',
      checkedLock: false,
      b_options:[],
      m_options:[],
      num: 1,
      bValue:'',
      mValue:'',
      dialogFormVisible: false,
      curPage: store.state.root.curPage,
      form: {
        name: '',
        pwd: ''
      },
      mySessionId : '',
      myUserName: store.state.root.userInfo.nickname,

      isLocked: false,
      formLabelWidth: '120px',
      files: [], //업로드용 파일
      filesPreview: [],
      uploadImageIndex: 0, // 이미지 업로드를 위한 변수

      OV: undefined,
      session: undefined,
      mainStreamManager: undefined,
      publisher: undefined,
      subscribers: [],

      block : false,
      mute : false,
      hostId : store.state.root.userInfo.userId,
      dialogImageUrl: '',
      dialogVisible: false,
      disabled: false,

      fileName: "파일을 올려주세요 (*.png, jpeg, jpg)",
      image: undefined,
    };
  },
  mounted() {
    this.$nextTick(function () {
      axios.get('category/book')
      .then((response) =>  {
        this.b_options = response.data;
      })
      .catch((error) => {
        console.log(error)
      })

      axios.get('category/movie')
      .then((response) =>  {
        this.m_options = response.data;
      })
      .catch((error) => {
        console.log(error);
      })
    })
  },
  methods: {
    chooseFile(){
      const fileInput = document.getElementById('file-input');
      fileInput.click();
    },
    getFile(e){
      const file = e.target.files[0];
      this.image = file;
      if(file){
        // this.image = file;
        const reader = new FileReader();
        reader.onload = function(){
          const result = reader.result;
          var uploadImage = document.getElementById("thumbnail-image");
          uploadImage.src = result;
          const wrapper = document.querySelector(".wrapper");
          wrapper.classList.add("active");
        }
        reader.readAsDataURL(file);
      }
      if(e.target.value){
        let valueStore = e.target.value.match(regExp);
        this.fileName = valueStore;
      }
    },
    removeImgAction(){
      var uploadImage = document.getElementById("thumbnail-image");
      uploadImage.src = "";
      const wrapper = document.querySelector(".wrapper");
      wrapper.classList.add("active");
      this.image = undefined;
    },
    fromClose1(){
      this.dialogFormVisible = false;
    },
    formClose() {
      this.dialogFormVisible = false;

      this.dynamicTags = ['키워드를', '입력하세요'];
      this.inputVisible = false
      this.inputValue = ''
      this.checkedLock = false
      this.num=1
      this.mySessionId=''
      this.bValue=''
      this.mValue=''
      //this.form.name=''
      this.isLocked=false
      this.form.pwd=''

      this.files = []
      this.filesPreview=[]
      this.uploadImageIndex=0;
    },

    handleClose(tag) {
        this.dynamicTags.splice(this.dynamicTags.indexOf(tag), 1);
    },

    showInput() {
      this.inputVisible = true;
      this.$nextTick(_ => {
        this.$refs.saveTagInput.$refs.input.focus();
      });
    },

    handleInputConfirm() {
      let inputValue = this.inputValue;
      if (inputValue) {
          this.dynamicTags.push(inputValue);
      }
      this.inputVisible = false;
      this.inputValue = '';
    },
    handleChange(value) {
        console.log(value);
    },
    handleCheckbox() {
        this.isLocked= !this.isLocked;
    },
    imageUpload() {
      console.log(this.$refs.files.files);

      // this.files = [...this.files, this.$refs.files.files];
      //하나의 배열로 넣기
      let num = -1;
      for (let i = 0; i < this.$refs.files.files.length; i++) {
        this.files = [
          ...this.files,
          //이미지 업로드
          {
              //실제 파일
              file: this.$refs.files.files[i],
              //이미지 프리뷰
              preview: URL.createObjectURL(this.$refs.files.files[i]),
              //삭제및 관리를 위한 number
              number: i
          }
        ];
        num = i;
        //이미지 업로드용 프리뷰
        // this.filesPreview = [
        //   ...this.filesPreview,
        //   { file: URL.createObjectURL(this.$refs.files.files[i]), number: i }
        // ];
      }
      this.uploadImageIndex = num + 1; //이미지 index의 마지막 값 + 1 저장
      console.log(this.files);
    },
    fileDeleteButton(e) {
      const name = e.target.getAttribute('name');
      this.files = this.files.filter(data => data.number !== Number(name));
    },
    //방생성 API
    async formRoom(){
      //console.log(this.files[0]);
      var data = {
        roomName : this.form.name,
        hostId: this.hostId,
        keywords : this.dynamicTags,
        limit: this.num,
        bookCategoryId:this.bValue,
        movieCategoryId: this.mValue,
        //"roomInviteCode": String,
        //password: String,
        thumbnailUrl: 'https://ifh.cc/g/FieTKm.png',
        //"email": email.value,
        password: this.form.pwd,
        sessionId: this.form.name,
      };
      if(this.image != undefined ){
        let frm = new FormData();
        frm.append('images', this.image);

        await axios.post('/images', frm,{ Headers: {'Content-Type': 'multipart/form-data'}})
        .then(res=>{
          data.thumbnailUrl = res.data;
          console.log(res.data);
        })
      }
      // axios({
      //   method:"POST",
      //   url: "rooms",
      //   data:{
      //     roomName : this.form.name,
      //     hostId: this.hostId,
      //     keywords : this.dynamicTags,
      //     limit: this.num,
      //     bookCategoryId:this.bValue,
      //     movieCategoryId: this.mValue,
      //     //"roomInviteCode": String,
      //     //password: String,
      //     thumbnailUrl: this.files[0],
      //     //"email": email.value,
      //     password: this.form.pwd,
      //     sessionId: this.form.name,
      //   },
      //   {
      //     header: { 'Content-Type': 'multipart/form-data' }
      // })

      await this.$store.dispatch(`root/createRoom`, data)
      .then((res) => {
        console.log(res.data)
        this.$router.push({ name : 'MeetingRoom', params: { roomName: this.form.name, keywords: this.dynamicTags }  })
        this.fromClose1()
      })
      .catch((err) => {
        if (err.response.status === 500) {
          alert('이미 존재하는 방 제목입니다. 다른 방 제목을 입력해주세요.')
        }
      });
    },
  }
}
</script>
<style>
@import "./CreateRoom.css";
@import url('https://fonts.googleapis.com/css?family=Poppins:400,500,600,700&display=swap');
*{
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Poppins', sans-serif;
}
.container{

  height: 350px;
  width: 100%;
  /* width: 430px; */
  position: relative;
}
.container .wrapper{
  position: relative;
  height: 300px;
  width: 100%;
  border-radius: 10px;
  background: #fff;
  border: 2px dashed #c2cdda;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}
.wrapper.active{
  border: none;
}
.wrapper .image{
  position: absolute;
  height: 100%;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}
.wrapper img{
  height: 100%;
  width: 100%;
  object-fit: cover;
}
.wrapper .icon{
  font-size: 100px;
  color: #9658fe;
}
.wrapper .text{
  font-size: 20px;
  font-weight: 500;
  color: #5B5B7B;
}
.wrapper #cancel-btn i{
  position: absolute;
  font-size: 20px;
  right: 15px;
  top: 15px;
  color: #9658fe;
  cursor: pointer;
  display: none;
}
.wrapper.active:hover #cancel-btn i{
  display: block;
}
.wrapper #cancel-btn i:hover{
  color: #e74c3c;
}
.wrapper .file-name{
  position: absolute;
  bottom: 0px;
  width: 100%;
  padding: 8px 0;
  font-size: 18px;
  color: #fff;
  display: none;
  background: linear-gradient(135deg,#3a8ffe 0%,#9658fe 100%);
}
.wrapper.active:hover .file-name{
  display: block;
}
.container #custom-btn{
  margin-top: 20px;
  display: block;
  width: 100%;
  height: 30px;
  border: none;
  outline: none;
  border-radius: 25px;
  color: #fff;
  font-size: 18px;
  font-weight: 500;
  letter-spacing: 1px;
  text-transform: uppercase;
  cursor: pointer;
  background: linear-gradient(135deg,#3a8ffe 0%,#9658fe 100%);
}
</style>
