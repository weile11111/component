<template>
  <div>
    <el-page-header
      style="line-height: 40px; margin-left: 20px"
      @back="goBack(devTranslateIP)"
      :content="'文件升级'"
    ></el-page-header>
    <el-dialog
      title="文件上传中..."
      :visible.sync="dialogProgressVisible"
      :before-close="handleClose"
      center
      width="40%"
      top="45vh"
    >
      <div class="conten">
        <el-progress
          :percentage="percentage"
          :color="colors"
          :status="progressStaus"
        ></el-progress>
      </div>
    </el-dialog>
    <div style="margin-right: 80px">
      <el-steps :active="3" align-center>
        <el-step title="选取本地文件"></el-step>
        <el-step title="选择上传设备"></el-step>
        <el-step title="选择上传设备，且不超过500M"></el-step>
      </el-steps>
    </div>
    <el-upload
      :limit="1"
      class="upload-demo"
      ref="upload"
      action=""
      :on-preview="handlePreview"
      :on-remove="handleRemove"
      :auto-upload="false"
      :on-change="beforeUpload"
      :file-list="fileList"
      style="margin-left: 100px; margin-top: 50px; margin-bottom: 50px"
    >
      <template>
        <el-button slot="trigger" size="small" type="primary"
          >选取文件
        </el-button>
        <div slot="tip" class="el-upload__tip" style="color: red">
          只能上传zip,rar,tar,tgz,bin或csv格式，且大小不超过500MB
        </div>
        <div slot="tip" class="el-upload__tip" style="color: red">
          将鼠标移动到文件上点X进行文件删除
        </div>
      </template>
    </el-upload>
    <div style="margin: 0 auto; width: 1000px">
      <el-main>
        <template>
          <el-transfer
            v-model="targetValue"
            :data="data"
            style="width: 680px; height: 200px; margin: 0 auto"
            :titles="['待选择机组', '已选择机组']"
          ></el-transfer>
        </template>
      </el-main>
      <el-col :span="6">
        <el-button
          style="margin-left: 10px; background-color: #03001e"
          size="small"
          type="success"
          @click="dialogVisible = true"
          >上传板卡文件到控制器
        </el-button>
      </el-col>
      <el-col :span="6">
        <el-button
          style="margin-left: 10px; background-color: #7303c0"
          size="small"
          type="success"
          @click="uploadFileToControl('model')"
          >上传模型文件到控制器
        </el-button>
      </el-col>
      <el-col :span="6">
        <el-button
          style="margin-left: 10px; background-color: #ec38bc"
          size="small"
          type="success"
          @click="uploadFileToControl('config')"
          >上传配置文件到控制器
        </el-button>
      </el-col>
      <el-col :span="6">
        <el-button
          style="margin-left: 10px; background-color: #fdeff9; color: black"
          size="small"
          type="success"
          @click="uploadFileToControl('system')"
          >上传系统文件到控制器
        </el-button>
      </el-col>
      <el-row style="margin-top:50px">
          <el-col :span="6">
            <el-button
              style="margin-left: 10px; background-color: #03001e"
              size="small"
              type="success"
              @click="uploadFileToControl('sftp')"
              >上传sftp文件到控制器
            </el-button>
          </el-col>
          <el-col :span="6">
            <el-button
              style="margin-left: 10px; background-color: #03001e"
              size="small"
              type="success"
              @click="uploadFileToControl('WCtable')"
              >上传工况表文件到控制器
            </el-button>
          </el-col>
      </el-row>
    </div>
    <table-result v-bind="dicrtData" ref="tableresult" v-if="tableResultVision">
    </table-result>
    <el-dialog
      title="请选择板卡"
      :visible.sync="dialogVisible"
      width="30%"
      :before-close="handleClose"
    >
      <el-radio v-model="mark" label="1">板卡1</el-radio>
      <el-radio v-model="mark" label="2">板卡2</el-radio>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button
          type="primary"
          @click="uploadFileToControl('card'), (dialogVisible = false)"
          >确 定</el-button
        >
      </span>
    </el-dialog>
  </div>
</template>

<script>
import SCP from "/src/main/scp/scp";
import { ModelAPI } from "@/api/mqtt";
import { setTimeout } from "timers";

export default {
  data() {
    const generateData = (_) => { 
      const data = [];
      for (let i = 1; i <= 8; i++) {
        data.push({
          key: i,
          label: `机组 ${i}`,
          disabled: i % 4 === 0,
        });
      }
      return data;
    };
    return {
      ip: process.env.SSH_IP_DEV2,
      dialogVisible: false,
      mark: "1",
      tempAs: {},
      end: false,
      resultSuccess: "success",
      dicrtData: {
        tableData: [],
        getData: [],
        title: "文件上传",
        status: "",
      },
      tableResultVision: false,
      selectDev: [],
      targetValue: [],
      data: [],
      Visible: false,
      progressStaus: null,
      dialogProgressVisible: false,
      percentage: 0,
      colors: [
        { color: "#f56c6c", percentage: 20 },
        { color: "#e6a23c", percentage: 40 },
        { color: "#6f7ad3", percentage: 60 },
        { color: "#1989fa", percentage: 80 },
        { color: "#5cb87a", percentage: 100 },
      ],
      filePath: [],
      fileList: [],
      fileInfo: [],
      business:"",
    };
  },

  mounted() {
    this.init();
  },

  //TODO:SFTP显示失败
  methods: {
    init(id, data) {
      let storage = window.sessionStorage;
      var result = storage.getItem("data");
      if (result === null || result === "") {
        this.$router.push("/table/index");
        this.ErrorMessage("无辅控设备，请检查网段刷新扫描");
      }
      this.Visible = true;
      this.$nextTick(() => {
        //获得机组编号
        var temp = [];
        let storage = window.sessionStorage;
        var connect = storage.getItem("data");
        JSON.parse(connect).forEach((element) => {
          temp.push({
            key: element.dev_ip,
            label: element.crewID,
            disabled: false,
          });
        });
        this.data = temp;
      });
    },

    /**
     * 描述 关闭文件操作
     * 参数：done 关闭确认
     *
     */
       handleClose(done) {
                this.$confirm("确认关闭？")
                .then((_) => {
                  done();
                })
                .catch((_) => {});
      },

    /**
     * 描述 回退操作
     * 参数：file 文件名
     *      fileList 文件表
     *
     */
    handleRemove(file, fileList) {
      this.filePath = [];
      console.log(file, fileList);
    },

     handlePreview(file) {
        console.log(file);
      },

    /**
     * 描述 获得文件路径
     * 参数：wind_field_id 风场id
     *
     */
    beforeUpload(file) {
      const fileSize = file.size / 1024 / 1024 < 500;

      if (!fileSize) {
        this.WarningMessage("文件大小不能超过500MB");
        this.fileList = [];
        return false;
      }

      this.filePath.push({ path: file.raw.path, name: file.name });
      console.log("文件路径", this.filePath);
    },

    uploadFileToControl(business) {
      if (this.filePath.length === 0) {
        this.WarningMessage("请选择文件!");
        return false;
      } else if (this.targetValue.length === 0) {
        this.WarningMessage("请选择机组!");
        return false;
      }
      this.percentage = 0;
      var fileNameIndex = this.filePath[0].name.lastIndexOf(".");
      var ext = this.filePath[0].name.substr(fileNameIndex + 1);
      console.log("ext", ext);
      //文件格式校验
      if (business == "model") {
        if (
          !(
            ext.indexOf("zip") != -1 ||
            ext.indexOf("rar") != -1 ||
            ext.indexOf("tar") != -1 ||
            ext.indexOf("tgz") != -1
          )
        ) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（zip,rar,tar,tgz）");
          return false;
        }
      }

      if (business == "system") {
        if (ext.indexOf("tgz") == -1) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（tgz)");
          return false;
        }
      }

      if (business == "card") {
        if (ext.indexOf("bin") == -1) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（bin）");
          return false;
        }
      }
      if (business == "config") {
        if (ext.indexOf("csv") == -1) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（csv）");
          return false;
        }
      }

      //新增sftp配置文件 txt格式
      if (business == "sftp") {
        if (ext.indexOf("txt") == -1) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（txt）");
          return false;
        }
      }


        //新增工况表配置文件 txt格式
      if (business == "WCtable") {
        if (ext.indexOf("csv") == -1) {
          this.filePath = [];
          this.fileList = [];
          this.WarningMessage("请上传正确类型的文件格式（csv）");
          return false;
        }
      }

      this.dialogProgressVisible = true;
      var uploadMessage = "";
      setTimeout(() => {
        (this.dicrtData = {
          tableData: [],
          getData: [],
          title: "文件上传",
          status: "",
        }),
          (this.resultSuccess = "");
        this.tableResultVision = false;
      }, 1);
      this.business = business;
      var uploadPath = "";
      this.$forceUpdate();
      if (business == "card") {
        uploadPath = process.env.MAIN_ADDRESS + "/mnt/kyimage/subcard/tmp";
        uploadMessage = "subcardupdateorrollback";
        this.taskCommand(uploadPath, uploadMessage);
      }
      if (business == "model") {
        this.submitUploadModel();
      }

      if (business == "config") {
        uploadPath = process.env.MAIN_ADDRESS + "/mnt/config/csv";
        uploadMessage = "testpointconfig";
        this.taskCommand(uploadPath, uploadMessage);
      }

      if (business == "system") {
        uploadPath = process.env.MAIN_ADDRESS + "/mnt/kyimage/assist/tmp";
        uploadMessage = "systemupdate";
        this.taskCommand(uploadPath, uploadMessage);
      }

      //新增sftp
      if (business == "sftp") {
        uploadPath = "/etc";
        uploadMessage = "sftpupdate";
        this.taskCommand(uploadPath, uploadMessage);
      }


      //新增工况表
      if (business == "WCtable") {
        uploadPath = process.env.MAIN_ADDRESS + "/mnt/config/csv";
        uploadMessage = "wctableupdate";
        // this.taskCommand(uploadPath, uploadMessage);
        this.asynUploadFileToControl(uploadPath, uploadMessage).then((result)=>{
          
        })
      }
    },

    taskCommand(uploadPath, uploadMessage) {
      Promise.all(
        this.asynUploadFileToControl(uploadPath, uploadMessage)
      )
        .then((result) => {
          this.log.info(result);
          var realyDb = this;

          setTimeout(() => {
            realyDb.percentage =
              realyDb.percentage + (1 / realyDb.targetValue.length) * 50;
          }, 1000);
          this.businessConplax();
        })
        .catch((err) => {
          this.businessConplax(err);
        });
    },

    /**
     * 描述 上传文件SSH命令
     * 参数：
     *
     */
    async asyncUploadSSH(index, uploadMessage) {
      return await new Promise((resolve, reject) => {
        var systemupdateStatus = {};
         systemupdateStatus = this.replayDBMessage({
            message: uploadMessage, 
            data: {
              host: this.targetValue[index],
              username: this.ip.username,
              password: this.ip.password,
              name: this.filePath[0].name,
              mark: this.mark,
            },
          });
        
        resolve(systemupdateStatus);
      });
    },

    /**
     * 描述 上传文件
     *
     * 参数：user 用户名
     * password 密码
     * host 主机名
     * filePath 文件路径
     * uploadPath 上传路径
     * callback 回调函数
     *
     */
    scpFun(names, host, filePath, uploadPath, callback) {
      const user = "root";
      if (names == "sftp") {
        SCP.scpUploadFile(
          filePath,
          uploadPath,
          host,
          user,
          this.ip.password,
          callback()
        );
      } else {
        SCP.scpUploadFile(
          filePath,
          uploadPath,
          host,
          this.ip.username,
          this.ip.password,
          callback()
        );
      }
    },

    /**
     * 描述 多上传文件
     * 参数：
     *      uploadPath 上传路径
     *      uploadMessage 上传信心
     *        
     */
    asynUploadFileToControl(uploadPath, uploadMessage) {
      var p_list = [];
      for (var index = 0; index < this.targetValue.length; index++) {
        p_list.push(
          new Promise((resolve, reject) => {
            console.log("当前主机数据:",this.targetValue[index]);
            var realyDb = this;
            console.log("文件路径", this.filePath[0].name);
            console.log("上传路径", uploadPath);
            console.log("地址", this.targetValue[index]);
            console.log("ip", this.ip);

            // 文件上传
            this.scpFun(
              this.business,
              this.targetValue[index],
              this.filePath[0].path,
              uploadPath,
              function (err) {
                console.log("回调信息" + index);
                if (err) {
                  this.ErrorMessage("上传文件失败", err);
                  reject(err);
                } else {
                    if(uploadMessage == "wctableupdate"){
                      this.SuccessMessage("工况表上传成功");
                      resolve(true);
                    }
                  // 进度条
                  setTimeout(() => {
                    realyDb.percentage =
                      realyDb.percentage +
                      (1 / realyDb.targetValue.length) * 50;
                  }, 200);

                  this.asyncUploadSSH(index, uploadMessage).then(
                     
                    (systemupdateStatus) => { 
                      setTimeout(() => {
                        try {
                          realyDb.dataHaved(systemupdateStatus, index,this.business);
                        } catch (error) {
                          realyDb.log.warn(error.message);
                          realyDb.progressStaus = "fail";
                          realyDb.percentage = 0;
                          console.log(typeof realyDb.dicrtData);
                          console.log(
                            "this。dat数据是：" + JSON.stringify(this.data)
                          );
                          realyDb.dicrtData.getData.push({
                            crewID: this.data[index - 1].label,
                            fileName: this.filePath[0].name,
                            status: (this.business=="sftp" || business=="WCtable")?"配置失败!辅控响应异常" : "升级失败！ 辅控响应异常",
                            ip: this.data[index -1].key,
                          });
                        }
                      
                      realyDb.fileList = [];
                      realyDb.filePath = [];
                      resolve("成功了！");
                    }
                    
                  );
                   }, 200);

                }
              }.bind(this)
            );
          })
        );
      }
      return p_list;
    },

    dataHaved(systemupdateStatus, index ,business) {
      var realyDb = this;
      if (systemupdateStatus.data.match("ok")) {
        realyDb.progressStaus = "success";
        realyDb.log.info(systemupdateStatus);
        console.log(typeof realyDb.dicrtData.getData);
        console.log("this。dat数据是：" + JSON.stringify(this.data));
        realyDb.dicrtData.getData.push({
          crewID: this.data[index - 1].label,
          fileName: this.filePath[0].name,
          status: (business=="sftp" || business=="WCtable")?"配置成功!":"升级成功！",
          ip: this.data[index - 1].key,
        });
      } else {
        realyDb.progressStaus = "fail";
        realyDb.percentage = 0;
        realyDb.log.warn("fail" + systemupdateStatus);
        console.log(typeof realyDb.dicrtData);
        console.log("this。dat数据是：" + JSON.stringify(this.data));
        realyDb.dicrtData.getData.push({
          crewID: this.data[index -1].label,
          fileName: this.filePath[0].name,
          status: (business=="sftp" || business=="WCtable")?"配置失败！" + "文件不兼容" : "升级失败！" + "升级文件不兼容",
          ip: this.data[index - 1].key,
        });
      }
    },

    /**
     * 描述 反馈页面
     * 参数 
     *      err 错误信息
     *
     */
    businessConplax(err) {
      var realyDb = this;
      var _default = {
        card: "板卡文件下装",
        config: "配置文件下装",
        system: "系统文件下装",
        "": "板卡文件下装",
        sftp: "sftp配置文件下装",
      };
      setTimeout(() => {
        realyDb.percentage = 0;
        realyDb.dialogProgressVisible = false;
        realyDb.dicrtData = {
          title: _default[this.business],
          status: err
            ? "fail"
            : realyDb.dicrtData.getData == "升级成功！"
            ? "success"
            : "fail",
          tableData: [
            { key: "crewID", label: "辅控IP" },
            { key: "fileName", label: "文件名" },
            { key: "status", label: "状态" },
          ],
          getData: realyDb.dicrtData.getData,
        };
        realyDb.$nextTick(() => {
          realyDb.tableResultVision = true;
        });
        const mainImg = realyDb.$refs.upload;
        if (mainImg && mainImg.length) {
          mainImg.forEach((item) => {
            // item.uploadFiles.length = 0;
            item.clearFiles();
          });
        }
      }, 1000);
    },

    /**
     * 描述 上传文件为了模型安装
     * 参数：crew_num 机组数量
     *
     */
    modelFileSuccess() {
      return new Promise((resolve, reject) => {
        const ip = process.env.SSH_IP_DEV2;

        var tempArr = [];
        var pathmodel = "/";
        this.targetValue.forEach((host, index) => {
          SCP.scpUploadFile(
            this.filePath[0].path,
            "/home/user/",
            host,
            ip.username,
            ip.password,
            function (err) {
              if (!err) {
                ModelAPI(
                  {
                    ip: host,
                    path:
                      process.env.MAIN_ADDRESS +
                      pathmodel +
                      this.filePath[0].name,
                  },
                  function (top, message) {
                    if (message.cmdresult == "success") {
                      this.progressStaus = "success";
                      if (message.badFlies.length) {
                        this.ErrorMessage(
                          "失败的模型为:" + message.badFlies.toString()
                        );
                        tempArr.push({
                          crewID: this.data[index].label,
                          fileName: this.filePath[0].name,
                          status:
                            "总共安装模型:" +
                            message.all +
                            "其中成功个数:" +
                            message.success +
                            "失败的模型为:" +
                            message.badFlies.toString(),
                        });
                      } else {
                        this.SuccessMessage(
                          "总共安装模型:" +
                            message.all +
                            "其中成功个数:" +
                            message.success
                        );
                        tempArr.push({
                          crewID: this.data[index].label,
                          fileName: this.filePath[0].name,
                          status:
                            "总共安装模型:" +
                            message.all +
                            "其中成功个数:" +
                            message.success,
                        });
                      }
                    } else if (message.cmdresult == "fail") {
                      this.ErrorMessage(message.reason);
                      tempArr.push({
                        crewID: this.data[index].label,
                        fileName: this.filePath[0].name,
                        status: "fail" + message.reason,
                      });
                    }
                    this.dialogProgressVisible = false;

                    this.filePath = [];
                    this.fileList = [];
                    this.dicrtData = {
                      title: "模型文件下装",
                      status: this.resultSuccess,
                      tableData: [
                        { key: "crewID", label: "SCADA机组编号" },
                        { key: "fileName", label: "文件名" },
                        { key: "status", label: "状态" },
                      ],
                      getData: tempArr,
                    };

                    this.tableResultVision = false;
                    this.tableResultVision = true;
                    this.percentage = 0;
                    resolve(message);
                  }.bind(this)
                );
              } else {
                console.log("失败原因", err);
                this.ErrorMessage("网络错误！");
                this.dialogProgressVisible = false;
              }
            }.bind(this)
          );
        });
      });
    },

    submitUploadModel() {
      setTimeout(() => {
        this.dicrtData = {};
        this.resultSuccess = "";
        this.tableResultVision = false;
      }, 1);
      this.$forceUpdate();
      this.percentage = this.percentage + (1 / this.targetValue.length) * 100;
      this.modelFileSuccess().then((message) => {
        console.log("d到这里了" + message);
      });
    },

    // /**
    //  * 描述 鉴定文件验交
    //  * 参数：wind_field_id 风场id
    //  *
    //  */
    // beforeAlbumUpload(file) {
    //   //必选选机组号
    //   const isJPG = file.type === "zip";
    //   const isLt2M = file.size / 1024 / 1024 < 2;

    //   if (!isJPG) {
    //     this.WarningMessage("上传图片只能是 zip格式!");
    //   }
    //   if (!isLt2M) {
    //     this.WarningMessage("上传图片大小不能超过 2MB!");
    //   }
    //   return isJPG && isLt2M;
    // },
 
  
    goBack() {
      this.$router.back(-1);
    },
  },
};
</script>
<style lang="scss">
.el-upload-list__item {
  width: 50%;
}
</style>
