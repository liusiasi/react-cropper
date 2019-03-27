# react-cropper
在React/dva框架下，使用react-cropper和antDesign的Upload组件实现的一个图片裁剪组件

# 一.主要的实现的功能
利用React组件化的思想封装一个裁剪并上传图片的组件，其中使用antDesign的Upload组件进行上传图片，并调用react-cropper组件实现固定裁剪框的长和宽的截图，将截图的结果以FILE格式上传给后台。后台返回图片的路径，显示在Upload组件上，并且可以删除和预览已经上传的图片。实现截图效果如下。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/first.png)

上传成功后可以通过Upload组件预览和删除，效果如下。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/second.png)
    
在一个广告管理的页面引用该组件，实现上传广告图片按照指定的尺寸的功能，广告管理页面如下。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/third.png)


# 二.首先安装react-cropper插件
npm install --save react-cropper
执行该命令以后，下载react-cropper依赖信息自动更新到package.json中
在使用该插件的代码中需要进行引入

import "cropperjs/dist/cropper.css"
import Cropper from 'react-cropper'

# 三.主要的思路
首先构建一个CropperUpload组件，该组件接收截图框的长，宽，Upload样式模式，初始图片列表四个参数。该组件主要实现上传可截图的图片的功能，在Upload组件的beforeUpload中用FileReader读取要上传的文件的base64码，并返回false阻止Upload组件自动上传，将文件码赋值给cropper组件，用cropper进行截图，截图之后，点击保存，触发saveImg函数，将cropper返回的base64码转换成File格式传向后台，并接收后台传回的图片路径，实现图片的预览。

# 四.具体的实现
组件CropperUpload的代码如下CropperUpload.js
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

引用CropperUpload的广告管理页面如下EidtBanner.js
```javascript
import React, { PureComponent } from 'react';
import { routerRedux, Route, Switch } from 'dva/router';
import { Spin, Card, Button, Form, Icon, Upload, Col, Row, DatePicker, TimePicker, Input, Modal, Select, Popover, Radio } from 'antd';
import moment from 'moment';
import { connect } from 'dva';
import PageHeaderLayout from '../../layouts/PageHeaderLayout';
import FooterToolbar from '../../components/FooterToolbar';
import styles from './SupplierElement.less';
import stylesDisabled from './ViewProduct.less';
import { message } from 'antd';
import CropperUpload from '../../components/Upload/CropperUpload';


const dateFormat = 'YYYY/MM/DD';
const { Option } = Select;
const { RangePicker } = DatePicker;
const FormItem = Form.Item;
const { TextArea } = Input;

const fieldLabels = {
  title: '标题',
  img: '广告图片',
  link: '广告内容图片',
  startTime: '上架时间',
  endTime: '下架时间',
  order: '排序',
  state: '状态',
  createDate: '创建时间',
  modifyDate: '编辑时间',
};

@connect(state => ({
  bannerobj: state.bannerDetail.bannerData.obj,
  detailLoading: state.bannerDetail.loading,
  success: state.addBannerData.changeData.success,
  msg: state.addBannerData.changeData.msg,
  submitLoading: state.addBannerData.loading,
}))
@Form.create()
export default class EidtBanner extends PureComponent {
  constructor(props) {
    super(props);
    this.state = {
      data: [],
      time: false,
      img: [],
      link: [],
      isEdit: true,
    }
  }
  componentDidMount() {
    const { dispatch } = this.props;
    const id = this.props.location.params.id;
    this.setState({
      isEdit: this.props.location.params.isEdit,
    });
    const data = { id };
    const values = Object.keys(data)
      .map(k => encodeURIComponent(k) + '=' + encodeURIComponent(data[k]))
      .join('&');
    dispatch({
      type: 'bannerDetail/getDetail',
      payload: values,
      callback: () => {
        let bannerdetail = this.props.bannerobj;
        if (bannerdetail.img) {
          let imageList = [];
          imageList.push({
            uid: -1,
            name: 'xxx.png',
            status: 'done',
            url: bannerdetail.img,
          });
          this.setState({
            img: imageList,
          });
        }

        if (bannerdetail.link) {
          let iconUrl = [];
          iconUrl.push({
            uid: -1,
            name: 'img.png',
            status: 'done',
            url: bannerdetail.link,
          });
          this.setState({
            link: iconUrl,
          });
        }
      },
    });
  }

  returnClick = () => {
    this.props.history.goBack();
  }

  validate = () => {
    const { form, dispatch, submitting } = this.props;
    const { getFieldDecorator, validateFieldsAndScroll, getFieldsError } = form;
    validateFieldsAndScroll((error, values) => {
      if (!error) {
        delete values.startTime;
        delete values.endTime;
        delete values.createDate;
        delete values.modifyDate;
        if (!this.state.img || this.state.img.length == 0) {
          message.error("请添加广告图片");
          return;
        }
        else {
          values.img = this.state.img[0].url;
        }
        if (!this.state.link || this.state.link.length == 0) {
          message.error("请添加广告内容图片");
          return;
        }
        else {
          values.link = this.state.link[0].url;
        }

        var data = Object.keys(values)
          .map(k => encodeURIComponent(k) + '=' + encodeURIComponent(values[k]))
          .join('&');
        dispatch({
          type: 'addBannerData/modifyBanner',
          payload: data,
          callback: () => {
            const { success, msg } = this.props;
            if (success === true) {
              message.success(msg);
              this.props.history.goBack();
            }
            else {
              message.error(msg);
            }
          },
        });
      }
    });
  }

  //广告图片的回调函数
  ChangeImage = (value) => {
    this.setState({
      img: value,
    })
  }
  //广告内容图片的回调函数
  ChangeBanner = (value) => {
    this.setState({
      link: value,
    })
  }
  render() {
    const { bannerobj, detailLoading } = this.props;
    const { isEdit } = this.state;
    const { getFieldDecorator, getFieldValue } = this.props.form;
    const { submitLoading } = this.props;
    const { fileList } = this.state;
    const uploadButton = (
      <div>
        <Icon type="plus" />
        <div className="ant-upload-text">Upload</div>
      </div>
    );
    const formItemLayout = {
      labelCol: { span: 6 },
      wrapperCol: { span: 6 },
    };
    const submitFormLayout = {
      wrapperCol: {
        xs: { span: 24, offset: 0 },
        sm: { span: 10, offset: 7 },
      },
    };
    const dateFormat = 'YYYY-MM-DD';
    return (
      <PageHeaderLayout>
        {
          detailLoading ? <div><Spin /></div> :
            <Form layout='horizontal'>
              <FormItem

              >
                {
                  getFieldDecorator('id', {
                    initialValue: bannerobj.id,
                  })(<Input type='hidden' />)
                }
              </FormItem>
              <FormItem {...formItemLayout} label={fieldLabels.title}>
                {getFieldDecorator('title', {

                  rules: [{ required: true, message: '请输入标题' }],
                  initialValue: bannerobj.title,
                })
                  (<Input placeholder="请输入标题" />)}
              </FormItem>

              <FormItem {...formItemLayout} label={fieldLabels.order}>
                {getFieldDecorator('order', {
                  rules: [{ required: true, message: '请输入排序' }],
                  initialValue: bannerobj.order,
                })
                  (<Input placeholder="请输入排序" />)}
              </FormItem>

              <FormItem {...formItemLayout} label={fieldLabels.startTime}>
                {getFieldDecorator('startTime', {
                  rules: [{ required: true, message: '请输入上架时间' }],
                  initialValue: moment(bannerobj.startTime).format('YYYY-MM-DD HH:mm:ss'),

                })(
                  <Input disabled={true} />
                )}
              </FormItem>

              <FormItem {...formItemLayout} label={fieldLabels.endTime}>
                {getFieldDecorator('endTime', {
                  rules: [{ required: true, message: '请输入下架时间' }],
                  initialValue: moment(bannerobj.endTime).format('YYYY-MM-DD HH:mm:ss'),

                })(
                  <Input disabled={true} />
                )}
              </FormItem>
              <FormItem {...formItemLayout} label={fieldLabels.createDate}>
                {getFieldDecorator('createDate', {
                  rules: [{ required: true, message: '请输入创建时间' }],
                  initialValue: moment(bannerobj.createDate).format('YYYY-MM-DD HH:mm:ss'),

                })(
                  <Input disabled={true} />
                )}
              </FormItem>

              <FormItem {...formItemLayout} label={fieldLabels.modifyDate}>
                {getFieldDecorator('modifyDate', {
                  rules: [{ required: true, message: '请输入编辑时间' }],
                  initialValue: moment(bannerobj.modifyDate).format('YYYY-MM-DD HH:mm:ss'),

                })(
                  <Input disabled={true} />
                )}
              </FormItem>
              <FormItem {...formItemLayout} label={fieldLabels.state}>
                {getFieldDecorator('state', {
                  rules: [{ required: true, message: '请选择状态' }],
                  initialValue: bannerobj.state,
                })(
                  <Select style={{ width: 100 }} >
                    <Option value={true}>有效</Option>
                    <Option value={false}>无效</Option>
                  </Select>
                )}
              </FormItem>


              <FormItem {...formItemLayout} label={fieldLabels.img}>
                <CropperUpload
                  pattern={2}//模式分为三种，1为只能上传一张图片，2为传一张图片，并且可以预览，3为传多张图片
                  width={375}
                  height={200}
                  onChange={this.ChangeImage.bind(this)}
                  fileList={this.state.img}
                />
              </FormItem>
              <FormItem {...formItemLayout} label={fieldLabels.link}>
                <CropperUpload
                  pattern={2}//模式分为三种，1为只能上传一张图片，2为传一张图片，并且可以预览，3为传多张图片
                  width={375}
                  height={200}
                  onChange={this.ChangeBanner.bind(this)}
                  fileList={this.state.link}
                />
              </FormItem>
              <FormItem {...submitFormLayout} className={styles.hideMarginBottom}>
                <Button type="primary" onClick={this.validate} loading={submitLoading} >提交</Button >

                <Button type="primary" style={{ marginLeft: 12 }} onClick={this.returnClick}>返回</Button >
              </FormItem>
            </Form>}
      </PageHeaderLayout >
    );
  }
}
```

# 五.实现中的细节可以见我的博客

[我的博客](https://blog.csdn.net/qq_36400206/article/details/88672171) 

