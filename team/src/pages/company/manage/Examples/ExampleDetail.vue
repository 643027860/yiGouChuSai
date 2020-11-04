<template>
  <div class="manageExampleDetail"
       v-loading="loading"
       element-loading-text="数据加载中"
       element-loading-spinner="el-icon-loading"
       element-loading-background="rgba(0, 0, 0, 0.3)">
    <el-dialog title="实例分析" :visible.sync="dialogExampleVisible" top="30vh" width="400px">
      <el-form ref="form"v-model="analyzeForm" label-width="80px">
        <el-form-item label="分析名称">
          <el-input v-model="analyzeForm.aeName" :fous="analyzeForm.errorTip = ''"></el-input>
        </el-form-item>
        <el-form-item label="模型算法">
          <el-select v-model="analyzeForm.mlAlgo" placeholder="请选择模型算法"  :fous="analyzeForm.errorTip = ''">
            <el-option label="Python" value="Python"></el-option>
            <el-option label="R" value="R"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="数据模型">
          <el-select v-model="analyzeForm.mlId" placeholder="请选择数据模型"  :fous="analyzeForm.errorTip = ''">
            <el-option
              v-for="model in modellist"
              :key="model.mlId"
              :label="model.mlName"
              :value="model.mlId">
            </el-option>
          </el-select>
        </el-form-item>
        <p style="text-align: center;margin: 0px;margin-bottom: 6px;color: red;" v-if="analyzeForm.errorTip">{{analyzeForm.errorTip}}</p>
        <el-form-item class="buttonItem">
          <el-button type="primary" @click="analyze()">立即分析</el-button>
          <el-button @click="dialogExampleVisible = false">取消</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
    <div class="headerSearchDiv">
      <el-form :inline="true" class="demo-form-inline">
        <el-form-item label="">
          <span style="font-size: 16px;margin-right: 20px;">数据实例详情：</span>
        </el-form-item>
      </el-form>
    </div>
    <div class="detailForm">
      <el-form ref="form" :model="qu" label-width="90px">
        <el-form-item label="实例名称：">
          {{qu.quName}}
        </el-form-item>
        <el-form-item label="生成时间：">
          {{formatTime(qu.quTime)}}
        </el-form-item>
        <el-form-item label="实例参数：" v-if="myDatas[0]">
          <el-tag
            v-for="(tag,index) in myDatas[0].nameData"
            :key="tag"
            v-show="tag"
          >
            {{tag}}
          </el-tag>
        </el-form-item>
        <el-form-item label="实例图示：">
          <div class="graphicalDiv formGraphicalDiv" v-for="(myData,index) in myDatas" :key="index">
            <div v-if="!myData.loading">
              <div class="titleDiv dataTitleDiv" >
                <p class="titleP" :id="index + '-' + myData.id">
                  {{myData.data.quName}}
                  <span class="titleIconSpan">
                    <el-button type="text" @click="toAnalyze(myData)">模型分析</el-button>
                  </span>
                </p>
              </div>
              <div class="contentDiv">
                <MyLine :MyLineId="myData.id" :myData="myData" v-if="myData.chartType=='折线图'"></MyLine>
                <MyPie :myPieId="myData.id" :myData="myData" v-if="myData.chartType=='饼图'"></MyPie>
                <MyBar :MyBarId="myData.id" :myData="myData" v-if="myData.chartType=='柱形图'"></MyBar>
              </div>
            </div>
          </div>
        </el-form-item>
      </el-form>
    </div>
  </div>
</template>

<script>
  import MyLine from '../../../../components/ChangeCharts/myLine'
  import MyPie from '../../../../components/ChangeCharts/myPie'
  import MyBar from '../../../../components/ChangeCharts/myBar'
  import {REQONEINTER,REQALLMODEL,REQONEMODEL,UPLOADANALYZE} from '../../../../api/types'
  import {mapState,mapActions} from 'vuex'
  export default {
    name: 'manageModelDetail',
    data () {
      return {
        qu:{},            //实例对象
        quId:'',          //实例Id

        myDatas:[],       //实例对象（图表）数组

        //数据分析表单
        analyzeForm:{
          mlAlgo:'Python',                  //模型算法
          aeName:'',                        //分析名称
          mlId:'',                          //模型ID
          quId:'',                          //实例ID
          qu:{},                            //实例
          model:{},                         //模型
        },
        dialogExampleVisible:false,         //是否弹出选择模型弹窗

        modelList:[],                       //模型列表
        modellist:[],                       //条件模型列表

        loading:false,
      }
    },
    components:{
      MyLine,MyPie,MyBar
    },
    methods: {
      ...mapActions(['choiceMenuId']),

      //查询实例，生成图表
      async reqExamplae(){
        var {quId} = this
        this.loading = true
        this.myDatas.push({loading:true})                                           //为了显示拉取数据中而加入的空数据
        var result = await REQONEINTER(quId)
        this.myDatas = this.myDatas.filter(myData => !myData.loading)               //去掉为了显示拉取数据中的空数据
        this.loading = false
        var id = new Date().getTime() + ''                                          //用于标识该表ID
        if(result && result.status == 0){
          this.qu = result.data
          var nameData = []
          //获取数据之后进行处理，横坐标放X轴，纵坐标放Y轴,y轴属性名放yNameData，x轴属性名放xNameData,chartType图表类型,quId列表查询ID,id改图标的id,data包含所有返回数据
          var myData = {data:result.data,x:[],y:[],yNameData:[],xNameData:[],nameData,chartType:'',quId,id}
          var qsVOMap = result.data.qsVOMap
          Object.keys(qsVOMap).forEach(key => {
            var isX = false
            myData.chartType = qsVOMap[key].chartType
            if(qsVOMap[key].chartOption == '横轴'){
              isX = true
            }
            if(!isX){
              myData.y.push(qsVOMap[key].fieldValues)
              myData.yNameData.push(key)
            }else{
              myData.x.push(qsVOMap[key].fieldValues)
              myData.xNameData.push(key)
            }
            myData.nameData.push(key)
          })
          this.myDatas.push(myData)
        }else{
          if(result.msg){
            var message = result.msg
          }else {
            var message = '查询实例失败'
          }
          this.message(message,'error',1500)
        }
      },

      //获取大数据模型列表,传入num的话就请求num页数据，否则请求第一页数据
      async reqModels(){
        var pageNum = 1
        var pageSize = 100
        var showFlag = 1
        var result = await REQALLMODEL({pageNum,pageSize,showFlag})
        if(result && result.status === 0){
          this.modelList = result.data
          this.selectModelList()
        }
      },

      //根据选择的算法筛选出对应的模型列表
      selectModelList(){
        var {modelList,analyzeForm} = this
        this.analyzeForm.mlId = ''
        this.modellist = modelList.modelListVOS.filter(item => analyzeForm.mlAlgo == item.mlAlgo)
      },

      //准备对实例进行分析
      toAnalyze(qu){
        this.analyzeForm.qu = qu         //保存需要进行分析的实例
        this.dialogExampleVisible = true //打开选择模型弹窗
      },
      //对实例进行分析
      async analyze(){
        var {mlId,qu,aeName} = this.analyzeForm
        if(mlId.trim() && qu.quId && aeName.trim()){
          //获取大数据模型详情
          this.loading = true
          this.dialogExampleVisible = false
          var result = await REQONEMODEL(mlId)
          if(result && result.status === 0){
            this.analyzeForm.model = result.data
            var {model} = this.analyzeForm
            var {msName,modelSmallVOS} = model
            var {quId} = qu
            var analyzeSmallDTOS = []
            qu.nameData.map(item => {
              if(item){
                modelSmallVOS.map(item1 => {
                  if(item1.msName === item){
                    analyzeSmallDTOS.push({msName:item,qsField:item})
                  }
                })
              }
            })
            if(analyzeSmallDTOS.length === qu.nameData.length){
              var result = await UPLOADANALYZE({aeName,analyzeSmallDTOS,mlId,quId})
              this.loading = false
              if(result && result.status === 0){
                this.message('分析成功','success',1500)
                this.empetyAnalyzeForm()
              }else{
                if(result.msg){
                  var message = result.msg
                }else{
                  var message = '数据分析失败'
                }
                this.message(message,'error',1500)
              }
            }else{
              this.loading = false
              this.message('实例模型不匹配','warning',1500)
            }
          }else{
            this.loading = false
            this.message('数据分析失败','error',1500)
          }
        }else{
          this.analyzeForm.errorTip = '请输入完整信息再进行分析'
        }
      },

      //返回规定格式的时间
      formatTime(time){
        if(time){
          return time.slice(0,10) + '    '+ time.slice(11,19)
        }
      },
      //清空选择模型表单内容
      empetyAnalyzeForm(){
        this.analyzeForm = {
          mlAlgo:'Python',
          aeName:'',
          mlId:'',
          quId:'',
          qu:{},
          model:{},
        }
      },
      //消息提示
      message(message,type,duration){
        this.$message({
          message,type,duration
        })
      }
    },

    created () {
      if(this.$route.query.quId){
        this.quId = this.$route.query.quId
      }else{
        this.$router.replace({name:'ExampleList'})
      }
    },
    mounted () {
      this.choiceMenuId('ExampleList')
      //查询数据实例
      this.reqExamplae()
      //获取模型列表
      this.reqModels()
    },
    watch: {
      'analyzeForm.mlAlgo'(newval,oldval){
        this.selectModelList()
      },
      dialogExampleVisible(newval,oldval){
        if(!newval){
          this.empetyAnalyzeForm()
        }
      }
    },
  }
</script>

<style lang="less">
  @import "../../../../assets/css/el-tag/el-tag";
  @import "../../../../assets/css/input/input";
  @import "../../../../assets/css/el-dialog/el-dialog";
  .manageExampleDetail{
    background-color: transparent;
    .headerSearchDiv{
      background-color: rgba(0,0,0,0.25);
      margin-bottom: 20px;
      padding: 10px 20px;
      .el-form{
        .el-form-item{
          margin: 0;
          margin-right: 10px;
        }
      }
      button{
        margin-right: 30px;
        padding: 10px 15px;
      }
    }
    button{
      outline: 0;
    }
    .detailForm{
      min-height: calc(100vh - 181px);
      padding: 20px;
      background-color: rgba(0,0,0,0.25);
      .el-form{
        .el-form-item{
          min-width: 500px;
          .el-form-item__label{
            color: white;
          }
          input,textarea{
            width: 40%!important;
            min-width: 600px;
          }
          .el-select{
            input{
              width: 100%!important;
            }
          }
          th{
            padding: 5px;
          }
          .myTable{
            width: 60%;
            min-width: 600px;
            input,textarea{
              border: 1px solid rgb(128,128,128)!important;
              width: 80%!important;
            }
          }
        }
      }
      .graphicalDiv{
        position: relative;
        height: 430px;
        margin-bottom: 23px;
        &:nth-last-child(1){
          margin-bottom: 0px;
        }
        background-color: rgba(0,0,0,0.25);
        border: 1px solid rgb(128,128,128);
        transition: 0.3s;
        .titleDiv{
          border-bottom: 1px solid rgb(128,128,128);
          .titleP{
            height: 48px;
            line-height: 48px;
            font-size: 16px;
            display: block;
            padding: 0 20px;
            margin: 0;
          }
          .titleIconSpan{
            display: inline-block;
            float: right;
            line-height: 48px;
            margin-right: 5px;
            button{
              font-size: 16px;
            }
          }
        }
        .contentDiv{
          height: 380px;
          padding: 20px;
        }
      }
      .emptyModelForm{
        height: 312px;
        border: 1px solid rgb(128,128,128);
        border-top: 0px;
        background-color: rgba(0,0,0,0.25);
      }
      .modelForm{
        background-color: rgba(0,0,0,0.25);
        padding: 10px 20px 30px;
        border: 1px solid rgb(128,128,128);
        border-top: 0px;
        .el-form-item__content{
          color: white;
        }
      }
    }
  }
  @media screen and(max-width: 767px){
    .manageExampleDetail{
      .detailForm{
        padding: 10px 0px 20px;
        .el-form{
          .el-form-item{
            min-width: 100%;
            margin-bottom: 10px;
            .el-form-item__label{
              color: white;
              padding: 0;
            }
            input,textarea{
              width: calc(100% - 10px);
              min-width: calc(100% - 10px);
            }
            .el-select{
              input{
                width: 100%!important;
              }
            }
            th{
              padding: 5px;
            }
            .myTable{
              width: calc(100% - 10px);
              min-width: calc(100% - 10px);
              input,textarea{
                border: 1px solid rgb(128,128,128)!important;
                width: 80%!important;
              }
            }
          }
        }
        .graphicalDiv{
          height: 383px;
          margin-bottom: 15px;
          .titleDiv{
            p{
              height: 20px;
              line-height: 20px;
              font-size: 14px;
            }
          }
          .contentDiv{
            height: 340px;
            padding: 10px;
          }
        }
        .formGraphicalDiv{
          margin-top: 40px;
          margin-left: -80px;
          margin-right: 10px;
        }
      }
    }
  }
</style>
