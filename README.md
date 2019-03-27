# react-cropper antDesign
在React/dva框架下，使用react-cropper和antDesign的Upload组件实现的图片裁剪插件
# 一.首先安装react-cropper插件
npm install --save react-cropper
执行该命令以后，下载react-cropper依赖信息自动更新到package.json中
在使用该插件的代码中需要进行引入

import "cropperjs/dist/cropper.css"
import Cropper from 'react-cropper'

# 二.主要的实现的功能
使用antdesign的upload组件进行上传图片，并调用react-cropper组件实现固定长和宽的截图，将截图的结果以FILE格式传给后台。实现效果如下。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/first.png)
可以上传多张图片。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/second.png)

# 三.主要的思路
首先构建一个CropperUpload组件，该组件接收截图框的长，宽，Upload样式模式，初始图片列表四个参数。该组件主要实现上传可截图的图片的功能，在Upload组件的beforeUpload中用FileReader读取要上传的文件的base64码，并返回false阻止Upload组件自动上传，将文件码赋值给cropper组件，用cropper进行截图，截图之后，将cropper返回的base64码转换成File格式传向后台。

# 四.具体的实现
组件CropperUpload的代码如下
```javascript
import React, { PureComponent } from 'react';
import moment from 'moment';
import { routerRedux, Route, Switch, Link } from 'dva/router';
import { Upload, Button, Modal, Icon, message } from 'antd';
import "cropperjs/dist/cropper.css"
import Cropper from 'react-cropper'
import { connect } from 'dva';
import styles from './Upload.less';

@connect(state => ({
  imagePictures: state.getData.imagePictures,
}))
class CropperUpload extends PureComponent {
  constructor(props) {
    super(props);
    this.state = {
      width: props.width,
      height: props.height,
      pattern: props.pattern,
      fileList: props.fileList ? props.fileList : [],
      editImageModalVisible: false,
      srcCropper: '',
      selectImgName: '',
    }
  }

  componentWillReceiveProps(nextProps) {
    if ('fileList' in nextProps) {
      this.setState({
        fileList: nextProps.fileList ? nextProps.fileList : [],
      });
    }
  }
  
  handleCancel = () => {
    this.setState({
      editImageModalVisible: false,
    });
  }

  // 特产介绍图片Upload上传之前函数
  beforeUpload(file, fileList) {
    const isLt10M = file.size / 1024 / 1024 < 10;
    if (!isLt10M) { //添加文件限制
      MsgBox.error({ content: '文件大小不能超过10M' });
      return false;
    }
    var reader = new FileReader();
    const image = new Image();
    var height;
    var width;
    //因为读取文件需要时间,所以要在回调函数中使用读取的结果
    reader.readAsDataURL(file); //开始读取文件
    reader.onload = (e) => {
      image.src = reader.result;
      image.onload = () => {
        height = image.naturalHeight;
        width = image.naturalWidth;
        if (height < this.state.height || width < this.state.width) {
          message.error('图片尺寸不对 宽应大于:'+this.state.width+ '高应大于:' +this.state.height );
          this.setState({
            editImageModalVisible: false, //打开控制裁剪弹窗的变量，为true即弹窗
          })
        }
        else{
          this.setState({
            srcCropper: e.target.result, //cropper的图片路径
            selectImgName: file.name, //文件名称
            selectImgSize: (file.size / 1024 / 1024), //文件大小
            selectImgSuffix: file.type.split("/")[1], //文件类型
            editImageModalVisible: true, //打开控制裁剪弹窗的变量，为true即弹窗
          })
          if (this.refs.cropper) {
            this.refs.cropper.replace(e.target.result);
          }
        }
      }
    }
    return false;
  }

  handleRemove(file) {
    this.setState((state) => {
      const index = state.fileList.indexOf(file);
      const newFileList = state.fileList.slice();
      newFileList.splice(index, 1);
      this.props.onChange(newFileList);
      return {
        fileList: newFileList,
      };
    });

  }

  //将base64码转化成blob格式
  convertBase64UrlToBlob(base64Data) {
    var byteString;
    if (base64Data.split(',')[0].indexOf('base64') >= 0) {
      byteString = atob(base64Data.split(',')[1]);
    } else {
      byteString = unescape(base64Data.split(',')[1]);
    }
    var mimeString = base64Data.split(',')[0].split(':')[1].split(';')[0];
    var ia = new Uint8Array(byteString.length);
    for (var i = 0; i < byteString.length; i++) {
      ia[i] = byteString.charCodeAt(i);
    }
    return new Blob([ia], { type: this.state.selectImgSuffix });
  }
  //将base64码转化为FILE格式
  dataURLtoFile(dataurl, filename) {
    var arr = dataurl.split(','),
      mime = arr[0].match(/:(.*?);/)[1],
      bstr = atob(arr[1]),
      n = bstr.length,
      u8arr = new Uint8Array(n);
    while (n--) {
      u8arr[n] = bstr.charCodeAt(n);
    }
    return new File([u8arr], filename, { type: mime });
  }

  _ready() {
    this.refs.cropper.setData({
      width: this.state.width,
      height: this.state.height,
    });
  }

  saveImg() {
    const { dispatch } = this.props;
    var formdata = new FormData();
    formdata.append("files", this.dataURLtoFile(this.refs.cropper.getCroppedCanvas().toDataURL(), this.state.selectImgName));
    dispatch({
      type: 'getData/postimage',
      payload: formdata,
      callback: () => {
        const { success, msg, obj } = this.props.imagePictures;
        if (success) {
          let imageArry = this.state.pattern == 3 ? this.state.fileList.slice() : [];
          imageArry.push({
            uid: Math.random() * 100000,
            name: this.state.selectImgName,
            status: 'done',
            url: obj[0].sourcePath,
            // url:obj[0].sourcePath,
            thumbUrl: this.refs.cropper.getCroppedCanvas().toDataURL(),
            thumbnailPath: obj[0].thumbnailPath,
            largePath: obj[0].largePath,
            mediumPath: obj[0].mediumPath,
            upload: true,
          })
          this.setState({
            fileList: imageArry,
            editImageModalVisible: false, //打开控制裁剪弹窗的变量，为true即弹窗
          })
          this.props.onChange(imageArry);
        }
        else {
          message.error(msg);
        }

      },
    });
  }
  render() {
    const botton = this.state.pattern == 2 ?
      <div>
        <Icon type="plus" />
        <div className="ant-upload-text">Upload</div>
      </div> :
      <Button>
        <Icon type="upload" />选择上传</Button>

    return (
      <div>
        <Upload
          name="files"
          action="/hyapi/resource/image/multisize/upload"
          listType={this.state.pattern === 1 ? " " : this.state.pattern === 2 ? "picture-card" : "picture"}
          className={this.state.pattern === 3 ? styles.uploadinline : ""}
          beforeUpload={this.beforeUpload.bind(this)}
          onRemove={this.handleRemove.bind(this)}
          fileList={this.state.fileList}
        >
          {botton}
        </Upload>
        <Modal
          key="cropper_img_icon_key"
          visible={this.state.editImageModalVisible}
          width="100%"
          footer={[
            <Button type="primary" onClick={this.saveImg.bind(this)} >保存</Button>,
            <Button onClick={this.handleCancel.bind(this)} >取消</Button>
          ]}>
          <Cropper
            src={this.state.srcCropper} //图片路径，即是base64的值，在Upload上传的时候获取到的
            ref="cropper"
            preview=".uploadCrop"
            viewMode={1} //定义cropper的视图模式
            zoomable={true} //是否允许放大图像
            movable={true}
            guides={true} //显示在裁剪框上方的虚线
            background={false} //是否显示背景的马赛克
            rotatable={false} //是否旋转
            style={{ height: '100%', width: '100%' }}
            cropBoxResizable={false}
            cropBoxMovable={true}
            dragMode="move"
            center={true}
            ready={this._ready.bind(this)}
          />
        </Modal>
      </div>
    );
  }
}

export default CropperUpload;
```
引用CropperUpload的页面如下
```javascript
import React, { PureComponent } from 'react';
import { routerRedux, Route, Switch } from 'dva/router';
import { Upload,Spin, Card, Button, Form, Icon, Col, Row, DatePicker, TimePicker, Input, Select, Popover } from 'antd';
import { connect } from 'dva';
import PageHeaderLayout from '../../../layouts/PageHeaderLayout';
import PartModal from '../../../components/Part/partModal';
import { message } from 'antd';
import stylesDisabled from './AddPart.less';
const { Option } = Select;
const { RangePicker } = DatePicker;
const FormItem = Form.Item;
import CropperUpload from '../../../components/Upload/CropperUpload';

const fieldLabels = {
  name: '标签名称',
  isActive: '状态',
};

const formItemLayout = {
  labelCol: {
    xs: { span: 24 },
    sm: { span: 7 },
  },
  wrapperCol: {
    xs: { span: 24 },
    sm: { span: 12 },
    md: { span: 5 },
  },
};

const submitFormLayout = {
  wrapperCol: {
    xs: { span: 24, offset: 0 },
    sm: { span: 10, offset: 7 },
  },
};

@connect(state => ({
  changeData: state.modifyDataCQX.changeData,
  submitLoading:state.modifyDataCQX.loading,
}))
@Form.create()
export default class LabelManageNew extends PureComponent {
  constructor(props) {
    super(props);
    this.state = {
      labelName: "",
      status: "",
      iconUrl:[],
    }
  }

  componentWillMount() {
    const { dispatch } = this.props;
    if (this.props.location.params) {
        this.setState({
          labelName: this.props.location.params.record.productName,
          status: this.props.location.params.record.isActive,
        });

      if(this.props.location.params.record.iconUrl){
        let iconUrl=[];
        iconUrl.push({
            uid: -1,
            name: 'iconUrl.png',
            status: 'done',
            url:this.props.location.params.record.iconUrl,
        });
        this.setState({
          iconUrl,
        });
    }
  }
  }

  returnClick = () => {
    this.props.history.goBack();
  }
  validate = () => {
    const { form, dispatch, submitting } = this.props;
    const { getFieldDecorator, validateFieldsAndScroll, getFieldsError } = form;
    validateFieldsAndScroll((error, values) => {
      if (!error) {
          let iconUrl='';
          if (this.state.iconUrl.length) {
            iconUrl=this.state.iconUrl[0].url;
          }
          else{
              message.error("请添加分区图标");
              return;
          }
          values.iconUrl=iconUrl;
          // var data = Object.keys(values)
          //   .map(k => encodeURIComponent(k) + '=' + encodeURIComponent(values[k]))
          //   .join('&');
          var data={
            ID:this.props.location.params.record.id,
            operatorName:this.props.location.params.record.operatorName,
            labelName:values.labelName,
            status:values.status,
            iconURL:values.iconUrl,
          }
          dispatch({
            type: 'modifyDataCQX/editLabelManagementFunc',
            payload: data,
            callback: () => {
              const { changeData  } = this.props;
              if (changeData.success === true) {
                message.success(changeData.msg);
                this.props.history.goBack();
              } else {
                message.error(changeData.msg);
              }
            },
          });
      }
    });
  }
  //标签内容图片的回调函数
  ChangeIconUrl = (value) => {
    this.setState({
      iconUrl: value,
    })
  }
  render() {
    const { form, dispatch } = this.props;
    const { submitLoading } = this.props;
    const { getFieldDecorator, validateFieldsAndScroll, getFieldsError } = form;
    return (
      <PageHeaderLayout >
        <Card bordered={false} title="标签信息">
          <Form hideRequiredMark className={stylesDisabled.disabled}>
            <FormItem {...formItemLayout} label={fieldLabels.name} >
              {getFieldDecorator('labelName', {
                rules: [{ required: true, message: '请输入标签名称' }],
                initialValue: this.state.labelName,
              })
                (<Input placeholder="请输入标签名称" />)
              }
            </FormItem>
            <FormItem {...formItemLayout} label={fieldLabels.isActive} >
              {getFieldDecorator('status', {
                rules: [{ required: true, message: '请输入状态' }],
                initialValue: this.state.status,
              })
                (<Select style={{ width: 100 }}>
                  <Option value={true}>有效</Option>
                  <Option value={false}>无效</Option>
                </Select>)
              }
            </FormItem>
            <FormItem  {...formItemLayout} label='标签图标'>
            <CropperUpload
                  pattern={1}//模式分为三种，1为只能上传一张图片，2为传一张图片，并且可以预览，3为传多张图片
                  width={375}
                  height={120}
                  onChange={this.ChangeIconUrl.bind(this)}
                  fileList={this.state.iconUrl}
                />
            </FormItem>
            <FormItem {...submitFormLayout} style={{ marginTop: 32 }}>
              <Button type="primary" onClick={this.validate} loading={submitLoading}>
                提交 </Button >
              < Button style={{ marginLeft: 8 }} onClick={this.returnClick} >
                返回
            </Button >
            </FormItem>
          </Form>
        </Card>
      </PageHeaderLayout>
    );
  }
}
```

# 五.实现中的细节见我的博客

[我的博客](https://blog.csdn.net/qq_36400206) 

