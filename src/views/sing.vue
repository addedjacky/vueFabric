<template>
  <div class="p-index">
        <div class="product-list" :class="$store.state.fabricObj.jigsawIsOpen ? 'open' : '' ">
            <top-bar ></top-bar>
            <guide></guide>
            <single-product :dataList="productList" :type="listType" @setSearchList="getSearchList"></single-product>
        </div>
        <div class="jigsaw" :class="$store.state.fabricObj.jigsawIsOpen ? 'open' : '' ">
            <div class="top-bar">
                <el-button type="text" @click="open" class="creare-case">{{caseBasic.title}}</el-button>
                 <el-button type="primary" class="save-btn" @click="saveCase()">保存案例</el-button>
            </div>
            <div class="opton"></div>
            <div class="black-board">
             <option-nav :config="config"></option-nav>
              <canvas id="canvas" width='761' height='589'></canvas>
            </div>
        </div>

  </div>

</template>
<script>
import topBar from "@/components/top_bar";
import guide from "@/components/guide";
import singleProduct from "@/components/single_product";
import optionNav from "@/components/option_nav";
import { create, updateCaseBasic, updateCanvas } from "@/api/case";
import { search } from "@/api/image";
import {qiniuToken,uploadToqiniu,getUrl} from "@/api/qiniu"
export default {
  data() {
    return {
      listType: "sing",
      productList: [],
      config: {
        canvasState: [],
        currentStateIndex: -1,
        redoStatus: false, //撤销状态
        undoStatus: false, // 重做状态
        undoFinishedStatus: 1,
        redoFinishedStatus: 1,
        undoButton: this.$refs.undo,
        redoButton: this.$refs.redo
      },
      caseBasic: {
        description: "",
        isEditable: "",
        isEditable: "",
        isPrivate: "",
        memberId: "",
        refId: "",
        thumb: "",
        title: "未命名"
      }
    };
  },
  components: { topBar, guide, singleProduct, optionNav },
  mounted() {
    this.$store.commit("setCurrentNav", "sing"); //绘制画布
    this.$store.commit("setOptionSelect", false); 
    this.updateImg();
    this.canvasDataChange(); //监听canvas 事件
    this.searchPdts(""); //搜索单品
  },
  methods: {
    updateImg() {
      this.$store.commit("setCanvas", this.fabricAction.createCanvas("canvas"));
    },
    firstBindEvent() {
      var _this = this;
      _this.$store.state.fabricObj.canvas.getObjects().forEach(function(k, i) {
        _this.fabricAction.bindSeletUnSelectEvent(k, _this);
      });
    },
    // canvas操作事件监听
    canvasDataChange() {
      let _self = this;
      this.$store.state.fabricObj.canvas.on("object:modified", function() {
       
        _self.updateCanvasState();
      });
      this.$store.state.fabricObj.canvas.on("object:added", function() {
        _self.updateCanvasState();
      });
      this.$store.state.fabricObj.canvas.on("object:removed", function() {
        _self.updateCanvasState();
      });
      this.$store.state.fabricObj.canvas.on("object:rotating", function() {
        _self.updateCanvasState();
      });
    },
    // 历史记录
    updateCanvasState() {
      var _self = this;
      if (
        _self.config.undoStatus == false &&
        _self.config.redoStatus == false
      ) {
        var jsonData = _self.$store.state.fabricObj.canvas.toJSON();
        var canvasAsJson = JSON.stringify(jsonData);
        if (
          _self.config.currentStateIndex <
          _self.config.canvasState.length - 1
        ) {
          var indexToBeInserted = _self.config.currentStateIndex + 1;
          _self.config.canvasState[indexToBeInserted] = canvasAsJson;
          var numberOfElementsToRetain = indexToBeInserted + 1;

          _self.config.canvasState = _self.config.canvasState.splice(
            0,
            numberOfElementsToRetain
          );
        } else {
          _self.config.canvasState.push(canvasAsJson);
        }
        _self.config.currentStateIndex = _self.config.canvasState.length - 1;
      }
    },

    /**@augments
     * fucntion 转为图片并下载到本地
     */
    canvasToImage() {
      var MIME_TYPE = "image/png";
      //转换成base64
      var imgURL = this.$store.state.fabricObj.canvas.toDataURL(MIME_TYPE); //创建一个a链接，模拟点击下载
      var dlLink = document.createElement("a");
      var filename = "个人画板_" + new Date().getTime() + ".png";
      dlLink.download = filename;
      dlLink.href = imgURL;
      dlLink.dataset.downloadurl = [
        MIME_TYPE,
        dlLink.download,
        dlLink.href
      ].join(":");
      document.body.appendChild(dlLink);
      dlLink.click();
      document.body.removeChild(dlLink);
    },
    open() {
      var _this = this;
      var html = [
        '  <div class="case-basic">',
        '     <div class="name-box">',
        '      <input class="case-name" id="caseTitle" value="' +
          this.caseBasic.title +
          '" />',
        "      </div>",
        "      <div>",
        "         <select><option>简约</option></select>",
        "         <select><option>客厅</option></select>",
        "      </div>",
        "     <div class='memo-box'><textarea id='caseMemo' class='area'>" +
          this.caseBasic.description +
          "</textarea></div>",
        "  </div>"
      ].join("");
      this.$alert(html, {
        dangerouslyUseHTMLString: true
      })
        .then(() => {
            _this.saveCase();
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已经取消"
          });
        });
    },
    searchPdts: function(keywords) {
      let name = this.$store.getters.images.currentClass.name;
      search(
        "k=" + keywords + "&c=" + (name == "" || name == undefined ? "" : name)
      ).then(
        res => {
          this.productList = res.data;
        },
        err => {}
      );
    },
    getSearchList(keywords) {
      this.searchPdts(keywords); //搜索单品
    },
    saveCase(){
      var _this=this;
      if(document.getElementById("caseTitle")&&document.getElementById("caseTitle").length!=0){
          _this.caseBasic.title = document.getElementById("caseTitle").value;
          _this.caseBasic.description = document.getElementById(
            "caseMemo"
          ).value;
      }
        if (!_this.caseBasic.paletteId) {
          create({
            description: _this.caseBasic.description, //描述
            isEditable: 0, //是否可以修改
            isPrivate: 0, //是否私有
            memberId: this.$store.state.user.userId, //会员ID
            refId: "a001", //引用ID
            thumb: "", // 缩略图url
            title: _this.caseBasic.title //案例主题或者名称吧
          }).then(res => {
            _this.caseBasic.memberId=this.$store.state.user.userId;
            _this.caseBasic.paletteId = res.data;
            var MIME_TYPE = "image/png";
            var _base64=_this.$store.state.fabricObj.canvas.toDataURL(MIME_TYPE);
            _this.uploadQiNiu(_base64);

          });
          return;
        }
        updateCaseBasic(_this.caseBasic).then(() => {
          let temp = this.$store.state.fabricObj.canvas.toObject();
          temp.paletteId = _this.caseBasic.paletteId;
          updateCanvas(temp);
        });
    },
    uploadQiNiu(base64Data){
        var domain ='';
          var _this=this;
          var  base64 =  base64Data.split("base64,")[1]
           let url = getUrl("thum" + new Date().getTime() + ".png");
          qiniuToken()//通过后台获取七牛云token
          .then((res)=>{
						//获取token成功后发送请求到七牛添加图片
						 domain = res.data.domain;
						let config={
							headers:{
								'Content-Type':'multipart/form-data',
								"Authorization":"UpToken "+res.data.uptoken
							}
            }
            return uploadToqiniu(url,base64,config)
            .then((res)=>{
          
              _this.caseBasic.thumb=domain+res.data.key;
    
              updateCaseBasic(_this.caseBasic).then(() => {
                let temp = this.$store.state.fabricObj.canvas.toObject();
                temp.paletteId = _this.caseBasic.paletteId;
                updateCanvas(temp);
              });
					})
					})
				
				
    }
  }
};
</script>

<style rel="stylesheet/scss" lang="scss">
/* @import url('../styles/test.scss'); */
.p-index {
  position: relative;
  overflow: hidden;
  background: #000;
  .save-btn {
    margin-left:10px;
    margin-top:10px;
  }
  .product-list {
    background: #fff;
    &.open {
      width: 36%;
      float: left;
    }
  }

  .jigsaw {
    display: none;
    position: relative;
    .bar-nav {
      background: #eee;
      padding: 10px;
      height: 30px;
      position: relative;
    }
    .optin-box {
      // display: none;
      position: relative;
      span {
        display: inline-block;
        width: 25px;
        margin-right: 20px;
        vertical-align: middle;
        float: left;
        &.option-special {
          width: 40px;
        }
        cursor: pointer;
        img {
          display: inline-block;
          float: left;
          width: 100%;
        }
      }
      .action {
        width: 220px;
        position: absolute;
        background-color: #eee;
        padding: 4px;
        top: 10px;
        left: 450px;
        border: #ccc 1px solid;
        display: none;
        &.select {
          display: block;
        }
        span {
          display: inline-block;
          img {
            width: 25px;
          }
        }
      }
    }
    .cut-optin {
      display: none;
      position: absolute;
      left: 100px;
      background: #fff;
      padding: 2px 10px;
      border: #eee 1px solid;
      span {
        display: inline-block;
        width: 25px;
        margin-right: 20px;
        cursor: pointer;
        img {
          display: inline-block;
          float: left;
          width: 100%;
        }
      }
    }
    .select {
      display: block;
    }

    &.open {
      display: block;
      float: left;
      width: 64%;
    }
    .top-bar {
      background: #363738;
      height: 64px;
    }
    .black-board {
      width: 800px;
      height: 1000px;
      background: #fff;
      position: fixed;
      left: 36%;
      margin-left: 80px;
      top: 100px;
    }
  }
  .creare-case {
    margin-left: 40px;
    text-decoration: none;
  }
}
.case-basic {
  .name-box {
    margin-bottom: 10px;
  }
  .case-name {
    width: 366px;
    border: #eee 1px solid;
    padding: 2px 10px;
    line-height: 20px;
  }
  select {
    width: 182px;
    margin-right: 10px;
  }
    .memo-box {
    margin-top: 10px;
    textarea {
      width: 386px;
      height: 60px;
      border: #eee 1px solid;
    }
  }
}
</style>


