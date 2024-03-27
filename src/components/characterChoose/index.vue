<template>
    <div class="contain">
      <div class="step">
        <el-steps :active="active" align-center>
          <el-step title="选择数据"></el-step>
          <el-step title="特征选择"></el-step>
          <el-step title="方法选择" v-if="type==4"></el-step>
          <el-step title="结果展示"></el-step>
        </el-steps>
      </div>
      <div class="mainCharacter">
        <div class="left_sidebar">
            <el-tree ref="tree"
            show-checkbox
            :check-strictly="true"  
            :data="treeData"
            :props="defaultProps"
            node-key="id"
            default-expand-all
            @check="handleCheck"
            @node-click="handleNodeClick">
          </el-tree>
        </div>
        <div class="right">
            <div class="info">
              <el-alert
                title="特征下方进度条为特征缺失度"
                type="info"
                center
                show-icon>
              </el-alert>
            </div>
            <br><br v-if="curentAnalyzeStep==1">            
            <h2 style="text-align: center" v-if="step==1">选择需要缺失补齐的特征</h2><h2 style="text-align: center" v-if="step==2" class="top">选择需要统计分析的特征</h2>
            <!--缺失值补齐，病人画像， 描述性分析的 特征选择-->
            <div class="selectClass" v-if="curentAnalyzeStep==1">
              <h3>文本类型字段</h3>
              <div class="contain3">
                <div class="block" v-for="feat in featureData" :key="feat.id" v-if="feat.featureDataType==3">
                    <el-checkbox v-model="feat.cheack" @change="handleCheckboxChange(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                    <!-- <el-progress :percentage="feat.missRate" :stroke-width=8 style="width: 100px;" class="custom-progress" ></el-progress> -->
                    <el-progress :percentage="feat.missRate" :stroke-width=8 :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
                <div class="block invisible" v-for="n in (5 - countVisibleFeatures(3) % 5) % 5" :key="n.id"></div>
              </div>
              <h3>连续数字类型字段</h3>
              <div class="contain1">
                <div class="block" v-for="feat in featureData" :key="feat.id" v-if="feat.featureDataType==1">
                  <el-checkbox v-model="feat.cheack" @change="handleCheckboxChange(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                  <!-- <el-progress :percentage="feat.missRate" :stroke-width=8 style="width: 120px;" class="custom-progress" :color="getProgressColor(feat.missRate)"></el-progress> -->
                  <el-progress :percentage="feat.missRate" :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
              <div class="block invisible" v-for="n in (5 - countVisibleFeatures(1) % 5 % 5) % 5" :key="n.id"></div>
              </div>
              <h3>离散数字类型字段</h3>
              <div class="contain2">
                <div class="block" v-for="feat in featureData" :key="feat.id" v-if="feat.featureDataType==2">
                    <el-checkbox v-model="feat.cheack" @change="handleCheckboxChange(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                    <!-- <el-progress :percentage="feat.missRate" :stroke-width=8 style="width: 120px;" class="custom-progress" :color="getProgressColor(feat.missRate)"></el-progress> -->
                    <el-progress :percentage="feat.missRate" :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
                <div class="block invisible" v-for="n in (5 - countVisibleFeatures(2) % 5% 5) % 5" :key="n.id"></div>
              </div>
            </div>
            <!--单因素分析的特征选择-->
            <div class="selectClassSingle" v-if="curentAnalyzeStep==2">
              <h3 style="text-align: center; font-size: 24px">选择作为分组依据特征</h3>
              <div class="contain4">
                <div class="block" v-for="(feat,index) in featureData" :key="index" v-if="feat!=observeFeaure && feat.discrete">
                    <el-checkbox v-model="feat.cheack" @change="handleCheckboxChangeGroup(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                    <el-progress :percentage="feat.missRate" :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
                <div class="block invisible" v-for="n in (5 - getGroupFeatLength() % 5) % 5" :key="n.id"></div>
              </div>
               <h3 style="text-align: center; font-size: 24px">选择需要观察的特征</h3>
              <div class="contain5">
                <div class="block" v-for="(feat,index) in featureData" :key="index" v-if="feat!=groupFeature">
                    <el-checkbox v-model="feat.cheack" @change="handleCheckboxChangeObserve(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                    <el-progress :percentage="feat.missRate" :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
                <div class="block invisible" v-for="n in (5 - getObserveFeatLength() % 5) % 5" :key="n.id"></div>
              </div>
            </div>

            <div class="consistencyAnalyze" v-if="curentAnalyzeStep==3">
               <h3 style="text-align: center; font-size: 24px">选择作为分组依据特征</h3>
               <div class="contain6">
                <div class="block" v-for="(feat,index) in featureData" :key="index">
                    <el-checkbox v-model="feat.cheack" @change="handleCheckboxChange(feat)" class="checkbox-font-size">{{feat.label}}</el-checkbox>
                    <el-progress :percentage="feat.missRate" :format="format" :color="getProgressStatus(feat.missRate)" style="width: 100px;"></el-progress>
                </div>
                <div class="block invisible" v-for="n in (5 - featureData.length % 5) % 5" :key="n.id"></div>
              </div>
            </div>
          </div>
      </div>
    </div>
</template>
<script>
import {getRequest, postRequest, getIndicators} from "@/api/user"
import { treeData } from "../tab/treeData";
/*特征选择页面*/
export default ({
    name:'characterChoose',
    props:['active','type', 'label','step','analyzeStep','describeAna'],
    data(){
        return{
          nodedata: {},
          selectedNode: [],
          treeData: [],
        defaultProps: {
          children: 'children',
          label: 'label'
        },
       options: [{
          value: '选项1',
          label: '全部特征'
        }, {
          value: '选项2',
          label: '疾病标准特征',
          disabled: false
        }, {
          value: '选项3',
          label: '非疾病标准特征'
        }],
        groupFeature: {},
        observeFeaure: {},
        allFeatures: [],
        curentAnalyzeStep: 1,
        checkedFeats: [],
        tableName: '',
        peopleFeatures: [],
        physiologicalFeatures: [],
        socialFeatures: [],
        checkAll_1: false,
        checkAll_2: false,
        checkAll_target: false,
        computeFeatures: [],
        knownFeatures: [],
        targetFeature: [],
        //用于存储左侧特征选择树值
        featureSelectTree: [],
        //特征选择数的备份，用于删除叶子节点
        filteredTree:[],
        featureData: [],
        //用于存储右侧特征条件选项
        feature_select_options: [
            { value: 0, label: "全部特征" },
            { value: 1, label: "疾病标准特征" },
            { value: 2, label: "非疾病标准特征" }
        ],
        //用于存储所有特征
        all_features: [],
        //用于存储已选的因变量（已选标签特征）
        checked_dependent_variables: [],
        //用于存储已选的因变量（已选危险因素）
        checked_independent_variables: [],
        //分页数据
        pageSize: 10,
        currentPage: 1,
        dataTotal: 10,
        value: '',
        templist: []
      }
    },
    created(){
      this.groupFeature= {};
      this.observeFeaure={};
      this.tableName = this.label;
      this.getIndicatorCategory();
      if(this.analyzeStep!=null){
        
        this.curentAnalyzeStep = this.analyzeStep;
      }
      
    },
    methods:{

      getProgressStatus(missingRate) {
      if (missingRate < 30) {
        return '#25bef5';
      } else if (missingRate < 70) {
        return '#e6a23c';
      } else {
        return '#ff0000';
      }

    },
    format(percentage) {
      return percentage === 100 ? '满' : `${percentage}%`;
    },

    getProgressColor(percentage) {
    if (percentage == 0 || percentage < 20) {
      return '#00cc66'; // 绿色
    } else if (percentage >= 20 && percentage < 50) {
      return '#3399ff'; // 蓝色
    } else if (percentage >= 50 && percentage < 80) {
      return '#ff6666'; // 红色
    } else {
      return '#cc0000'; // 深红色
    }
  },
      getGroupFeatLength(){
        let count = 1;
        for(let i=0; i<this.featureData.length; i++) {
          if (this.featureData[i].discrete) count+=1;
        }
        if(this.observeFeaure!={}) {
          return count-1;
        }
         else {
          return count;
         }
      },
      getObserveFeatLength(){
        if(this.groupFeature!={}) return this.featureData.length-1;
        else return this.featureData.length;
      },
      handleCheckboxChangeGroup(feat){
        if(feat.cheack) { // 其他都设置为非选中状态
          this.groupFeature = feat;
          // 发送给父组件
          this.$emit("send_group_feature",feat);
          for(let i=0; i<this.featureData.length; i++){
            if(this.featureData[i]!=feat && this.featureData[i]!=this.observeFeaure) this.featureData[i].cheack = false;
          }
        }else{
          this.groupFeature = {};
        }
      },
      handleCheckboxChangeObserve(feat){
        if(feat.cheack){
          // 发送给父组件
          this.observeFeaure = feat;
          this.$emit("send_observe_feature",feat);
          for(var i=0; i<this.featureData.length; i++){
            if(this.featureData[i]!=this.groupFeature && this.featureData[i]!=feat) this.featureData[i].cheack = false;
          }
        }else{
          this.observeFeaure = {};
        }
      },
      countVisibleFeatures(val) {
        return this.featureData.filter(feat => feat.featureDataType == val).length;
      },
      handleCheckboxChange(feat){
        console.log("状态变化：")
        console.log(feat);
        if(this.checkedFeats.includes(feat)){
          if(!feat.cheack) { // 非选中状态
            // if(this.describeAna==true) this.checkedFeats=[];
            // else this.checkedFeats = this.checkedFeats.filter(item => item !== feat);
            this.checkedFeats = this.checkedFeats.filter(item => item !== feat);
          }
        }else{
          if(feat.cheack) { // 选中状态
            // if(this.describeAna==true){ // 描述性分析只能选一个
            //   this.checkedFeats = [];
            //   for(let i=0; i<this.featureData.length; i++){
            //     if(this.featureData[i]!=feat) this.featureData[i].cheack = false;
            //   }
            // }
            this.checkedFeats.push(feat);
          }
        }
        // 传递给父组件
        this.$emit("send_feat",this.checkedFeats)
      },
      getIndicatorsFromBackEnd(types){
        console.log("tableName  "+this.tableName)
        getIndicators("/api/getIndicators",types, this.tableName).then(response=>{
          console.log("features:",response.data)
          this.featureData = response.data;
          // 给父组件传递参数
          this.$emit("send_indicators",this.featureData)
        }).catch(error=>{
        })
      },
      handleCheck(data,node){
        this.selectedNode.length = 0;
        for(let i=0; i<node.checkedNodes.length; i++){
          this.selectedNode.push(node.checkedNodes[i].label);
        }
        this.getIndicatorsFromBackEnd(this.selectedNode.join(","));
      },
      handleNodeClick(item,node,self){ //自己定义的editCheckId，防止单选出现混乱
        this.nodedata = node.data;
        this.editCheckId=item.id;
        this.$refs.tree.setCheckedKeys([item.id])
      },
      getIndicatorCategory(){
        getRequest("/api/indicatorCategory").then(response=>{
          this.treeData = response.data;
        })
      }
    },
    mounted(){
    }

})

</script>
<style lang="less">
// .title{
//   margin-left: 0px;
//   margin-bottom: 0px;
//   font-size: 15px; /* 修改字体大小 */
// }

.mainCharacter {
    display: flex;
    width: 100%;
    border-top: 1px solid black;
    margin-top: 20px;
    height: 750px;
}
.top{
  front-size: 20px;
  margin-top: 0px;
}
.contain {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: auto;
}
 .contain1 {
    display: flex;
    width: 100%;
    flex-direction: row; /* 按列排版 */
    height: calc(30% - 20px);/* 每个容器宽度为父容器的1/3，减去间距 */
    padding: 10px;
    flex-wrap: wrap; /* 每行换行 */

  }

   .contain2 {
    display: flex;
     width: 100%;
    flex-direction: row; /* 按列排版 */
    height: calc(30% - 20px); /* 每个容器宽度为父容器的1/3，减去间距 */
    padding: 10px;
    flex-wrap: wrap; /* 每行换行 */
  }
   .contain3 {
    display: flex;
    width: 100%;
    flex-direction: row; /* 按列排版 */
    height: calc(28% - 20px);; /* 每个容器宽度为父容器的1/3，减去间距 */
    padding: 10px;
    overflow-y: auto;
    flex-wrap: wrap; /* 每行换行 */
  }

  .contain4 {
    display: flex;
    width: 100%;
    flex-direction: row; /* 按列排版 */
    height: calc(40% - 20px);; /* 每个容器宽度为父容器的1/3，减去间距 */
    margin-top: 15px;
    margin-bottom: 5px;
    overflow-y: auto;
    flex-wrap: wrap; /* 每行换行 */
  }
  .contain5 {
    display: flex;
    width: 100%;
    flex-direction: row; /* 按列排版 */
    height: calc(40% - 20px);; /* 每个容器宽度为父容器的1/3，减去间距 */
    margin-top: 15px;
    margin-bottom: 15px;
    overflow-y: auto;
    flex-wrap: wrap; /* 每行换行 */
  }

  .contain6{
    display: flex;
    width: 90%;
    flex-direction: row;
    margin-top: 30px;
    margin-bottom: 5px;
    height: 70%;
    overflow-y: auto;
    flex-wrap: wrap;

  }

.step{
    width: 100%;
    text-align: center;
}
.left_sidebar {
    height: 700px;
    flex: 2;
    padding: 5x;
    margin-top: 20px;
    box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
}
.right{
    flex: 12;
    display: flex;
    flex-direction: column;
    box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
    height: 720px;
}


.info {
  margin: 0; /* 移除默认的外边距 */
  height: 30px;
  margin-left: 100px;
  margin-right: 100px;
  display: flex;
  // background-color: #cbc8c8;
}

.selectClass{
    margin-top: 0px;
    margin-left: 100px;
    width: 1300px;
    height: 800px;
}
.selectClassSingle{
    margin-top: 10px;
    margin-left: 100px;
    width: 1300px;
    height: 800px;
}
.consistencyAnalyze{
    margin-top: 10px;
    margin-left: 100px;
    width: 1300px;
    height: 800px;
}
.block {
    flex: 20%; /* 每个块占据1/6的行宽 */
    height: 70px;
    align-items: center; 
}

.block2 {
    flex: 16.66%; /* 每个块占据1/6的行宽 */
    height: 70px;
    align-items: center; 
}
.checkbox-font-size .el-checkbox__label {
    font-family: "宋体", SimSun; /* 设置字体为宋体 */
    font-size: 20px; /* 设置您希望的字体大小 */
    font-weight: bold; /* 加粗字体 */
}



.table{
  // display: flex;
  margin-top: 50px;
  width: 1400px;
}


.feature {
  display: block; 
  height: auto;
  width: 1400px;
  margin-left: 20px;
}
.checkbox-font-size .el-checkbox__inner {
  border-color: #0066cc; /* 设置边框颜色 */
  border-width: 2px; /* 增加边框宽度 */
}


.progress-green {
  background-color: #00cc66; /* 绿色 */
}

.progress-blue {
  background-color: #3399ff; /* 蓝色 */
}

.progress-red {
  background-color: #ff6666; /* 红色 */
}

.progress-dark-red {
  background-color: #cc0000; /* 深红色 */
}



</style>