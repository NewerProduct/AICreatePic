<script setup lang="ts">
import axios from "axios";
import type { FormInstance, InputInstance, FormRules } from "element-plus";
import { ElInput, ElInputNumber, ElMessage, ElLoading } from "element-plus"
import { reactive, ref, nextTick, } from "vue";
import CryptoJS from 'crypto-js';
import { onBeforeMount } from "vue";

//定义要执行校验的表单
const rules = reactive<FormRules<formType>>({
  init_images: [
    { required: true, message: '请上传照片', trigger: 'blur' },
  ]
})
interface formType {
  init_images: string[];
  sd_model_checkpoint: string;
  refBd: boolean;
  refHair: boolean;
  //使用索引签名 来限制 所有键必须是字符串类型，这样可以动态的向该对象添加属性名和属性值。
  prompt: { [index: string]: number };
  steps: number;
  cfg_scale: number;
  sampler_index: string;
  seed: number;
  negative_prompt: { [key: string]: number };
  pictureNum: number;
  denoising_strength: number;
  width: number;
  height: number;
  mask: null,
  mask_blur: null,
  inpainting_mask_invert: null
}

const form = reactive<formType>({
  init_images: [],
  sd_model_checkpoint: '',
  refBd: false,
  refHair: false,
  prompt: {},
  steps: 20,
  cfg_scale: 1,
  sampler_index: 'DPM++ 2M',
  seed: -1,
  negative_prompt: {
    "Easy negative": 1.0,
    "nsfw": 1.0,
    "lowres": 1.0,
    "bad anatomy": 1.0,
    "bad hands": 1.0,
    "text": 1.0,
    "error": 1.0,
    "missing fingers": 1.0,
    "extra digit": 1.0,
    "fewer digits": 1.0,
    "cropped": 1.0,
    "worst quality": 1.0,
    "low quality": 1.0,
    "normal quality": 1.0,
    "jpeg artifacts": 1.0,
    "signature": 1.0,
    "watermark": 1.0,
    "username": 1.0,
    "blurry": 1.0
  },
  pictureNum: 1,
  denoising_strength: 0.4,
  width: 512,
  height: 512,
  mask: null,
  mask_blur: null,
  inpainting_mask_invert: null
})
const defaultForm: formType = {
  init_images: [],
  sd_model_checkpoint: '',
  refBd: false,
  refHair: false,
  prompt: {},
  steps: 20,
  cfg_scale: 1,
  sampler_index: '',
  seed: -1,
  negative_prompt: {
    "Easy negative": 1.0,
    "nsfw": 1.0,
    "lowres": 1.0,
    "bad anatomy": 1.0,
    "bad hands": 1.0,
    "text": 1.0,
    "error": 1.0,
    "missing fingers": 1.0,
    "extra digit": 1.0,
    "fewer digits": 1.0,
    "cropped": 1.0,
    "worst quality": 1.0,
    "low quality": 1.0,
    "normal quality": 1.0,
    "jpeg artifacts": 1.0,
    "signature": 1.0,
    "watermark": 1.0,
    "username": 1.0,
    "blurry": 1.0
  },
  pictureNum: 1,
  denoising_strength: 0.4,
  width: 512,
  height: 512,
  mask: null,
  mask_blur: null,
  inpainting_mask_invert: null
};
//定义返回来的照片的url
const resultImg = ref('')
//尺寸选项，由于绑定的是对象，需要用来value-key来保证唯一性,这里选择创建数组 id是唯一的。
const sizeOptions = [
  { id: 1, label: "原始比例", width: 512, height: 512 },
  { id: 2, label: "2:3", width: 512, height: 768 },
  { id: 3, label: "4:3", width: 512, height: 384 },
];
//定义响应式对象来接取查询返回来的数据
let dataArr:any = ref([]);

//定义响应式对象来接收查询返回来的图片的各种数据
let picDataArr:any = ref({});

//单一的提示词和反提示词只用来收集数据，不需要提交，这里提取出来另做存储
const needLessSubmit = reactive({
  tipStrength: 1,
  reverseTipWord: '',
  tipWord: '',
  //这里默认值设为原始比例的值，即可成功设置默认选中原始比例。
  size: {
    id: 1, label: "原始比例", width: 512, height: 512
  }
})
//创建表单的引用 - 第一个表单
const formRef1 = ref<FormInstance>()
const formRef2 = ref<FormInstance>()
const formRef3 = ref<FormInstance>()




const inputVisible = ref(false)
const reverseInputVisible = ref(false)
//提示词引用
const InputRef = ref<InputInstance>()
//反提示词引用
const ReverseInputRef = ref<InputInstance>()

//定义调取摄像头功能
let video = ref()   // 获取video的DOM

let dialogVisible = ref()
//为了实现前端显示中文，表单数据携带应为提示词，这里需要做一个翻译映射
const translationMap = new Map<string,string>();
//定义要用到的参数
const appid = '20230707001736522';
const sKey = '4tFGUp8nIK2bh06PpdX_';
//接入百度翻译api
async function baiduTranslateEN(word: string) {
  //定义翻译后的结果
  let t_Result = ref('');
  //生成salt
  const salt = new Date().getTime();
  //生成签名
  const sign = appid + word + salt + sKey;
  const signStr = sign.replace(/-/g, ''); // 移除签名中的'-'
  const md5Sign = CryptoJS.MD5(signStr).toString(); // MD5加密签名
  try {
    //利用axios向翻译发送请求
    const result = await axios({
      url: "/fanyi",
      method: 'POST',
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      data: {
        q: word,
        from: 'zh',
        to: 'en',
        appid: appid,
        salt: salt,
        sign: md5Sign
      }
    })
    //成功后打印输出结果
    console.log(result);
    const { data } = result
    const { trans_result } = data;
    t_Result.value = trans_result[0].dst
    
    //将翻译的结果添加进映射里面
    const englishWord = t_Result.value
    console.log("2222",englishWord);
    console.log(word);
    
    translationMap.set(englishWord,word);
    //返回翻译的结果
    return englishWord;
  } catch (error: any) {
    //抛出一个错误提示
    throw new Error(error.message)
  }
}

// 调起摄像头
const handleOpen = async () => {
  dialogVisible.value = true
  try {
    // 调起摄像头
    const stream = await navigator.mediaDevices.getUserMedia({ video: true })
    // 也可以调摄像头的比例
    // const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 300,height: 400 }})
    const videoElement = video.value
    // 将摄像头画面绑定到 video 元素上
    videoElement.srcObject = stream
    videoElement.play()
  } catch (error) {
    console.error('无法访问摄像头:', error)
  }
}

// 点击确认
let photo = ref()   // 获取img的DOM
const submitImg = () => {
  dialogVisible.value = false
  // 创建一个canvas
  const canvas = document.createElement('canvas')
  canvas.width = video.value.videoWidth
  canvas.height = video.value.videoHeight
  // 将摄像头捕捉到的图像绘制到canvas上
  canvas.getContext('2d')!.drawImage(video.value, 0, 0, canvas.width, canvas.height)
  // 将canvas转换为图片，这时候的图片格式是base64，如果需要什么格式，需要另外进行转换 - canvas.toDataURL('image/png')
  //同时也赋值给表单的init_images属性
  form.init_images.push(canvas.toDataURL('image/png'))
  //从表单当中拿去照片的base64值
  photo.value.src = form.init_images[0];
  console.log("1111", form);

  // 如果使用formData的话
  //  let formData = new FormData()
  //  formData.append('picture', photo.value.src)
  // 如果使用formdata，那么接口需要添加 headers: {'Content-Type': 'multipart/form-data'} 
  // 显示图片
  photo.value.style.display = 'block'
  clearVideo()
}

const clearVideo = () => {
  // 关闭摄像头
  const stream = video.value.srcObject
  const tracks = stream.getTracks()
  tracks.forEach((track: any) => {
    track.stop()
  })
  video.value.srcObject = null
}
//关闭对话框时，也关闭摄像头
function closeCamera() {
  clearVideo()
}



const handleClose = (tag: string, isReverse: boolean = false) => {
  if (isReverse) {
    delete form.negative_prompt[tag];
  } else {
    delete form.prompt[tag];
  }
}

const showInput = (isReverse: boolean = false) => {
  if (isReverse) {
    reverseInputVisible.value = true
    nextTick(() => {
      ReverseInputRef.value!.input!.focus()
    })
  } else {
    inputVisible.value = true
    nextTick(() => {
      InputRef.value!.input!.focus()
    })
  }
}

const handleInputConfirm = async (isReverse: boolean = false) => {
  if (isReverse) {
    if (needLessSubmit.reverseTipWord) {
      //接入翻译函数
      const value = await baiduTranslateEN(needLessSubmit.reverseTipWord);
      form.negative_prompt[value] = needLessSubmit.tipStrength;
    }
    reverseInputVisible.value = false
    needLessSubmit.reverseTipWord = ''
  } else {
    if (needLessSubmit.tipWord) {
      //接入翻译函数
      const value = await baiduTranslateEN(needLessSubmit.tipWord);
      form.prompt[value] = needLessSubmit.tipStrength;
    }
    inputVisible.value = false
    needLessSubmit.tipWord = ''
  }
}
//提交表单
const onSubmit = async () => {
  if (form.init_images.length === 0) {
    ElMessage({ message: "请先上传照片", type: 'error' })
    return;
  }
  //调用加载服务
  const loadingObj = ElLoading.service({
    text: '正在绘图中，请稍等....',
    fullscreen: true,
  })
  const formattedPrompt = formatPrompt(form.prompt);
  const formattedNegativePrompt = formatPrompt(form.negative_prompt);

  // 创建一个新的对象，包含格式化后的 prompt 和 negative_prompt - 该表单是最后提交的表单。
  const submissionData = {
    ...form,
    prompt: formattedPrompt,
    negative_prompt: formattedNegativePrompt,
  };

  console.log("提交的数据:", submissionData);
  let result;
  try {
    //向处理图片的服务器发送请求
    result = await axios({
      method: "POST",
      url: "/sdapi/v1/img2img",
      //携带的参数，即是表单的数据
      data: submissionData,
    })
    //解构出data
    const { data } = result;
    console.log(result);

    //赋值给url
    resultImg.value = data.images[0];
  } catch (error: any) {
    //当出现错误时，停止加载动画 - 同时抛出错误提示框
    loadingObj.close()
    ElMessage({
      message: error.message,
      type: 'error',
    })
  }
  if (result) {
    //成功获取数据后关闭后提示成功消息。
    loadingObj.close()
    ElMessage({
      message: '绘图完成。',
      type: 'success',
    })
  }
  // //清空表单。
  // onReset();
};

//重置表单
const onReset = () => {
  // 重置所有表单字段为默认值
  Object.assign(form, defaultForm);

  // 重置 prompt 和 negative_prompt
  form.prompt = {};
  form.init_images = [];

  // 重置输入可见性
  inputVisible.value = false;
  reverseInputVisible.value = false;

  //清空图片
  photo.value.src = ' ';
  //清空生成后的图片
  resultImg.value = '';
  console.log("表单已重置");
}

function changeStyle(event: any) {
  if (event.target.tagName === 'BUTTON') {
    const btn = document.querySelector('.btn.cover');
    if (btn) btn.classList.remove("cover")
    event.target.classList.add('cover')
  }
}

const updateStrength = (tag: string, value: number, isReverse: boolean = false) => {
  if (isReverse) {
    form.negative_prompt[tag] = value;
  } else {
    form.prompt[tag] = value;
  }
}
//获取选中选择的value值
function getSize(value: any) {
  form.width = value.width
  form.height = value.height
}
//格式化数据函数，用来生成格式为(提示词:权值) 数据
function formatPrompt(promptObj: { [key: string]: number }):string {
  return Object.entries(promptObj)
    .map(([key, value]) => `(${key}:${value.toFixed(1)})`)
    .join(',');
}


//添加一个新的函数，来map映射中获取中文部分
function getChineseWord(englishWord:string):string {
  //利用get方法，传入键来获取对应的值
   const result = translationMap.get(englishWord)
  if(result) {
    return result;
  }else {
    return englishWord;
  }
}
//发送请求，获得最底部遍历的图片数据
async function getPictureData(promptName:string){
  const result = await axios({
    url:'http://localhost:9520/api/Aicavas/title',
    method:'GET',
    params:{
      title:promptName
    }
  })
  //结构出图片数据
  const {data} = result
  picDataArr.value = data.data.prompt
  // console.log('1111',data);
  
  // const {prompt} = data

  //获得后端传回来提示词内容。
  // picDataArr.value = prompt;
  console.log(picDataArr.value);
  console.log(JSON.parse(picDataArr.value));
  const tgObj = JSON.parse(picDataArr.value);
  //对格式化的数据进行操作。
  tgObj.forEach((element:any) => {
    //把数据存入map中
    translationMap.set(element.name,element.CNname)
    //向form中追加新数据
    form.prompt[element.name] = element.value
  });
 


  
}
//在接入一个获取全部预设列表的请求。
async function getAllLists(){
  const result = await axios({
    url:'http://localhost:9520/api/Aicavas/allLists',
    method:'GET',
  })
  console.log(result);
  const {data:{data}}=result;
  //拿到由对象组成的数组。
  dataArr.value = data
  console.log(data);
  
}
onBeforeMount(()=>{
  //挂载之后调用函数
  getAllLists();
})
</script>

<template>
  <div class="outer">
    <div class="header">
      <div class="image">
        卿云AI画布
      </div>
      <div class="nav" @click="changeStyle($event)">
        <button class="btn cover">AI写真</button>
        <button class="btn">背景重绘</button>
      </div>
    </div>

    <div class="htmlBody">
      <div class="midcontainer">
        <div class="container-left">
          <div class="left">
            <div>
              <img style="display: none" ref="photo" />
              <el-button type="info" 
              style="width: 400px;
              height: 200px; 
              margin-top: 10px;
              border-radius: 20px;"
               @click="handleOpen">打开相机</el-button>
              <!-- 弹窗 -->
              <el-dialog v-model="dialogVisible" title="照片预览" width="500" @closed="closeCamera">
                <div>
                  <video ref="video" width="500" height="500" muted></video>
                </div>
                <el-button type="info" @click="submitImg">拍照</el-button>
              </el-dialog>
            </div>
          </div>
          <div class="right">
            <span>
              图片生成区
            </span>
            <div v-if="resultImg === ''">
              <div class="someTip">
                <el-icon>
                  <WarningFilled />
                </el-icon>
                <span>温馨提示</span>
                <article style="font-size:18px;">打开相机成功拍下照片后，请点击开始生成按钮即可获得处理后的照片</article>
              </div>
            </div>
            <div v-else>
              <img :src="'data:image/png;base64,' + resultImg" alt="生成后的图片" style="margin: 0  auto;">
            </div>
          </div>
        </div>
      </div>
      <div class="container-right">
        <div class="partOne">
          <el-form ref="formRef1" :model="form" label-width="auto" style="max-width: 600px" id="form" :rules="rules">
            <el-form-item class="formItem" label="基础模型" style="display: block;">
              <el-select placeholder="请选择你要使用的模型" class="el-select" v-model="form.sd_model_checkpoint" clearable>
                <el-option label="动漫风格" value="anything-v4.0-pruned-fp16.safetensors [d5f8393fea]" />
                <el-option label="写实风格" value="majicmix7.safetensors [7c819b6d13]" />
              </el-select>
            </el-form-item>
            <el-form-item class="formItem" label="提示词" style="display: block;" prop="tipWord">
              <div class="tipWord">
                <div class="tipContent">
                  <div class="flex gap-2">
                    <el-tag v-for="(strength, tag) in form.prompt" :key="tag" closable :disable-transitions="false"
                      @close="handleClose(tag)">
                      <el-tooltip class="box-item" effect="dark" placement="top-start">
                        <template #content>
                          <div>
                            强度:
                            <el-input-number v-model="form.prompt[tag]" :min="0" :max="2" :step="0.1"
                              @change="(value) => updateStrength(tag, value)" />
                          </div>
                        </template>
                        <el-button class="tagButton">
                          <div class="tipStrength">
                            <div>{{ strength.toFixed(1) }}</div>
                          </div>
                          {{ getChineseWord(tag) }}
                        </el-button>
                      </el-tooltip>
                    </el-tag>
                    <el-input v-if="inputVisible" ref="InputRef" v-model="needLessSubmit.tipWord" class="w-20"
                      size="small" @keyup.enter="handleInputConfirm()" @blur="handleInputConfirm()"
                      input-style="width:50px;" />
                    <el-button v-else class="button-new-tag" size="small" @click="showInput()">
                      + 提示词
                    </el-button>
                  </div>
                </div>
              </div>
            </el-form-item>
          </el-form>
        </div>
        <div class="partTwo">
          <el-menu>
            <el-sub-menu index="1" :popper-offset="8">
              <template #title>高级设置</template>
              <el-form ref="formRef2" :model="form">
                <el-menu-item index="1-1">
                  <el-form-item label="生成步数" label-style="width:50px; margin-top:50px;">
                    <el-input type="text" v-model="form.steps" />
                  </el-form-item>
                </el-menu-item>
                <el-menu-item index="1-2" style="margin-bottom:20px">
                  <el-form-item label="提示词引导系数">
                    <div class="slider-demo-block">
                      <el-slider v-model="form.cfg_scale" show-input :show-input-controls="false" size="small"
                        :max="30" />
                    </div>
                  </el-form-item>
                </el-menu-item>
                <el-menu-item index="1-3">
                  <el-form-item label="采样器">
                    <el-select v-model="form.sampler_index" style="width: 235px;" placeholder="请选择采样器">
                      <el-option label="DPM++ 2M" value="DPM++ 2M" />
                    </el-select>
                  </el-form-item>
                </el-menu-item>
                <el-menu-item index="1-4">
                  <el-form-item label="随机种子">
                    <el-input type="text" v-model="form.seed" :input-style="{ width: '200px' }" />
                  </el-form-item>
                </el-menu-item>
                <el-menu-item index="1-5" class="rTip">
                  <el-form-item label="反向提示词" prop="reverseTipWord">
                    <div class="tipWord">
                      <div class="tipContent">
                        <div class="flex gap-2">
                          <el-tag v-for="(strength, tag) in form.negative_prompt" :key="tag" closable
                            :disable-transitions="false" @close="handleClose(tag, true)">
                            <el-tooltip class="box-item" effect="dark" placement="top-start">
                              <template #content>
                                <div>
                                  强度:
                                  <el-input-number v-model="form.negative_prompt[tag]" :min="0" :max="2" :step="0.1"
                                    @change="(value) => updateStrength(tag, value, true)" />
                                </div>
                              </template>
                              <el-button class="tagButton">
                                <div class="tipStrength">
                                  <div>{{ strength.toFixed(1) }}</div>
                                </div>
                                {{ getChineseWord(tag) }}
                              </el-button>
                            </el-tooltip>
                          </el-tag>
                          <el-input v-if="reverseInputVisible" ref="ReverseInputRef"
                            v-model="needLessSubmit.reverseTipWord" class="w-20" size="small"
                            @keyup.enter="handleInputConfirm(true)" @blur="handleInputConfirm(true)"
                            input-style="width:50px;" />
                          <el-button v-else class="button-new-tag" size="small" @click="showInput(true)">
                            + 反提示词
                          </el-button>
                        </div>
                      </div>
                    </div>
                  </el-form-item>
                </el-menu-item>
              </el-form>
            </el-sub-menu>
          </el-menu>
        </div>
        <div class="partThree">
          <el-menu>
            <el-sub-menu index="1" :popper-offset="8">
              <template #title>生成设置</template>
              <el-form ref="formRef3" :model="form">
                <el-menu-item index="1-1">
                  <el-form-item label="尺寸" style="margin-top: 15px">
                    <el-select type="text" v-model="needLessSubmit.size" placeholder="请选择尺寸" @change="getSize"
                      value-key="id">
                      <el-option v-for="option in sizeOptions" :key="option.id" :label="option.label"
                        :value="option"></el-option>
                    </el-select>
                  </el-form-item>
                </el-menu-item>
              </el-form>
            </el-sub-menu>
          </el-menu>
        </div>
        <div class="sp">

          <el-button @click="onReset()" class="resetBtn"><el-icon :size="20">
              <RefreshRight />
            </el-icon></el-button>
          <el-button class="sparkle-button" @click="onSubmit()">
            <span class="spark"></span>

            <span class="backdrop"></span>
            <svg class="sparkle" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path
                d="M14.187 8.096L15 5.25L15.813 8.096C16.0231 8.83114 16.4171 9.50062 16.9577 10.0413C17.4984 10.5819 18.1679 10.9759 18.903 11.186L21.75 12L18.904 12.813C18.1689 13.0231 17.4994 13.4171 16.9587 13.9577C16.4181 14.4984 16.0241 15.1679 15.814 15.903L15 18.75L14.187 15.904C13.9769 15.1689 13.5829 14.4994 13.0423 13.9587C12.5016 13.4181 11.8321 13.0241 11.097 12.814L8.25 12L11.096 11.187C11.8311 10.9769 12.5006 10.5829 13.0413 10.0423C13.5819 9.50162 13.9759 8.83214 14.186 8.097L14.187 8.096Z"
                fill="black" stroke="black" stroke-linecap="round" stroke-linejoin="round"></path>
              <path
                d="M6 14.25L5.741 15.285C5.59267 15.8785 5.28579 16.4206 4.85319 16.8532C4.42059 17.2858 3.87853 17.5927 3.285 17.741L2.25 18L3.285 18.259C3.87853 18.4073 4.42059 18.7142 4.85319 19.1468C5.28579 19.5794 5.59267 20.1215 5.741 20.715L6 21.75L6.259 20.715C6.40725 20.1216 6.71398 19.5796 7.14639 19.147C7.5788 18.7144 8.12065 18.4075 8.714 18.259L9.75 18L8.714 17.741C8.12065 17.5925 7.5788 17.2856 7.14639 16.853C6.71398 16.4204 6.40725 15.8784 6.259 15.285L6 14.25Z"
                fill="black" stroke="black" stroke-linecap="round" stroke-linejoin="round"></path>
              <path
                d="M6.5 4L6.303 4.5915C6.24777 4.75718 6.15472 4.90774 6.03123 5.03123C5.90774 5.15472 5.75718 5.24777 5.5915 5.303L5 5.5L5.5915 5.697C5.75718 5.75223 5.90774 5.84528 6.03123 5.96877C6.15472 6.09226 6.24777 6.24282 6.303 6.4085L6.5 7L6.697 6.4085C6.75223 6.24282 6.84528 6.09226 6.96877 5.96877C7.09226 5.84528 7.24282 5.75223 7.4085 5.697L8 5.5L7.4085 5.303C7.24282 5.24777 7.09226 5.15472 6.96877 5.03123C6.84528 4.90774 6.75223 4.75718 6.697 4.5915L6.5 4Z"
                fill="black" stroke="black" stroke-linecap="round" stroke-linejoin="round"></path>
            </svg>
            <span class="text">开始生成</span>
          </el-button>

        </div>

      </div>
      <div class="footer">
        <!-- 写一个盒子，装着照片和标题吗，利用v-for进行遍历即可。 -->
        <div class="list" v-for="(item,key) in dataArr" 
        :key="key"
        @click="getPictureData(item.title)"
        >
          <img src="https://staticsns.cdn.bcebos.com/amis/2024-7/1722395652477/damo.png" alt="大漠奇遇">
          <div class="bottom">
            <span>{{item.title }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
@import './index.css';
@import './buttonStyle/button.css';
</style>
