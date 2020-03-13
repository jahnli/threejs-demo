<template>
  <div id="app">
    <div id="container" style="width: 800px;height: 600px"></div>
    <el-button type="primary" @click="splitHandle">{{isOpen ? '收起':'展开'}}</el-button>
    <div class="label" v-show="showTooltip">{{tooltipText}}</div>
  </div>
</template>
<script>
  import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer'
  import {OBJLoader,MTLLoader} from 'three-obj-mtl-loader';
  import * as THREE from 'three';
  import 'three-orbitcontrols';
  import {Vector2,Raycaster,MorphAnimation,AxisHelper,PointLight,RectAreaLight,PointLightHelper,AmbientLight,WebGLRenderer,PerspectiveCamera,Scene,Mesh,MeshPhongMaterial,SpotLight,SpotLightHelper,DirectionalLight,MeshBasicMaterial,MeshLambertMaterial,DoubleSide} from 'three'
  export default {
    data(){
      return{
        camera: null,
        scene: null,
        renderer: null,
        mesh: null,
        mouse:null,
        labelRenderer:null,
        raycaster:null,
        clickObjects:[],
        warn:'1-2',
        serverObjs:[],
        doorObjs:[],
        switchObjs:[],
        rdhxObjs:[],
        allObjs:[],
        isOpen:false,
        showTooltip:false,
        tooltipText:''
      }
    },
    mounted() {
      let arr = ['1-1','2-3','3-2','1-4','5-4','3-2','2-1','2-2','2-3','2-4','5-2','8-2','7-1','7-4','6-4','6-2']
      this.init();
      this.animate()
      setInterval(()=>{
        let random = Math.floor(Math.random() * 15 + 1)
        this.warn = arr[random];
        this.$notify({
          title: '警告',
          message: `Server-${this.warn} 温度过高`,
          type: 'warning'
        });

      },3000)
      setInterval(()=>{
        // this.splitHandle()
      },1000)
    },
    methods: {
      // 初始化
      init() {
        const dom = document.querySelector('#container');
        this.mouse = new Vector2();
        this.raycaster = new Raycaster();
        // 初始化场景
        this.scene = new Scene();
        // 初始化渲染器
        this.initRenderer(dom)
        // 初始化相机
        this.initCamera(dom);
        // 初始化灯光
        this.initLight()
        // 初始化坐标辅助
        // this.initAxis()
        // 加载地板
        this.initGroud(dom);
        this.initControl();
        // 加载模型
        this.initMesh(dom);

        this.labelRenderer = new CSS2DRenderer();
        this.labelRenderer.setSize( dom.clientWidth, dom.clientHeight );
        this.labelRenderer.domElement.style.position = 'absolute';
        this.labelRenderer.domElement.style.top = '0';
        this.labelRenderer.domElement.style.pointerEvents = 'none';
        document.getElementById( 'container' ).appendChild( this.labelRenderer.domElement );
      },
      // 初始化模型
      initMesh(dom){
        let mtlLoader = new MTLLoader();
        mtlLoader.load('/rack.mtl', (materials) =>{
          // 预加载
          materials.preload();
          const loader = new OBJLoader();//在init函数中，创建loader变量，用于导入模型
          loader.setMaterials( materials )
          loader.load('/rack.obj', (obj)=> {
            //第一个表示模型路径，第二个表示完成导入后的回调函数，一般我们需要在这个回调函数中将导入的模型添加到场景中
            obj.traverse((child) =>{
              if (child instanceof Mesh ){
                if(child.name.includes('sever')){
                  this.serverObjs.push(child)
                }
                if(child.name == 'CDU'){
                  this.addLabel(child,'CDU',[0,2,0],[-0.3,2.5,0],[-0.8,2.5,0],[-.5,2.6,0])
                }
                if(child.name == 'PDU'){
                  this.addLabel(child,'S-ICRC',[0.1,1,-0.4],[-0.3,0.8,-0.4],[-1,0.8,-0.4],[-.5,0.9,-0.4])
                }
                if(child.name == 'pdu'){
                  this.addLabel(child,'SPDU',[-0.3,1,0.4],[-0.5,0.8,0.4],[-1,0.8,0.4],[-.5,0.9,0.4])
                }
                if(child.name.includes('door') || child.name.includes('EMO')|| child.name.includes('minipc')){
                  this.doorObjs.push(child)
                  if(child.name == 'EMO'){
                    this.addLabel(child,'EMO',[0.2,1.5,-.5],[1,2.5,-.5],[1.5,2.5,-.5],[1.5,2.6,-.5])
                  }
                  if(child.name == 'minipc'){
                    this.addLabel(child,'MiniPC',[0,1.5,-.5],[1,0.5,-.5],[1.5,0.5,-.5],[1.5,0.6,-.5])
                  }
                }
                if(child.name.includes('left') || child.name.includes('right')|| child.name.includes('switch')){
                  this.switchObjs.push(child)
                  if(child.name == 'switch3'){
                    this.addLabel(child,'Switch',[0.2,1,1],[1,1.5,1],[1.5,1.5,1],[1.5,1.6,1])
                  }
                  if(child.name == 'switch1'){
                    this.addLabel(child,'Switch',[-0.2,1,1],[-1,1.5,1],[-1.5,1.5,1],[-1.1,1.6,1])
                  }
                }
                if(child.name.includes('rdhx')){
                  this.rdhxObjs.push(child)
                  this.addLabel(child,'RDHx',[-0.2,2,1.2],[-1,2.5,1.2],[-1.5,2.5,1.2],[-1.1,2.6,1.2])
                }
                child.material.side = DoubleSide;
                this.allObjs = [...this.serverObjs,...this.doorObjs,...this.switchObjs,...this.rdhxObjs];
              }
            });
            dom.addEventListener('mousedown', this.onMouseDown, false);
            obj.position.y = -1
            this.mesh = obj;//储存到全局变量中
            this.scene.add(obj);//将导入的模型添加到场景中
          });
        })
      },
      // 添加标注
      addLabel(child,text,p1,p2,p3,textPosition){
        // 添加线
        const lineGeometry = new THREE.Geometry();
        const lineMaterial = new THREE.LineBasicMaterial({vertexColors:true});
        const color = new THREE.Color('#009FFF');
        let _p1 = new THREE.Vector3(...p1);
        let _p2 = new THREE.Vector3(...p2);
        let _p3 = new THREE.Vector3(...p3);
        lineGeometry.vertices.push(_p1);
        lineGeometry.vertices.push(_p2);
        lineGeometry.vertices.push(_p2);
        lineGeometry.vertices.push(_p3);
        lineGeometry.colors.push(color,color,color,color)
        const line = new THREE.Line(lineGeometry,lineMaterial,THREE.LineSegments);
        child.add(line);
        // 创建文字
        const textLoader = new THREE.FontLoader();
        textLoader.load( 'helvetiker_regular.typeface.json',  ( font ) => {
          const textgeometry = new THREE.TextGeometry( text, {
            font: font,
            size: .1,
            height:0.01,
          } );
          const textMaterial = new THREE.MeshBasicMaterial({
            flatShading: THREE.FlatShading,
            transparent: true,
            color:'#009FFF'
          });
          const textMesh = new THREE.Mesh(textgeometry, textMaterial);
          textMesh.position.set(...textPosition)
          textMesh.rotateY(135)
          child.add(textMesh)
        });
      },
      onMouseDown(event){
        event.preventDefault();
        this.mouse.x = (event.clientX / this.renderer.domElement.clientWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / this.renderer.domElement.clientHeight) * 2 + 1;
        this.raycaster.setFromCamera(this.mouse, this.camera);
        //总结一下，这里必须装网格，mesh，装入组是没有效果的
        //所以我们将所有的盒子的网格放入对象就可以了
        // 需要被监听的对象要存储在clickObjects中。
        const intersects = this.raycaster.intersectObjects(this.allObjs);
        const dom = document.querySelector('.label');
        const tooltipObj = this.scene.getObjectByName('tooltip');
        if(intersects.length > 0) {
          // 在这里填写点击代码
          let item = intersects[0].object;
          this.showTooltip = true;
          if(!tooltipObj){
            const labelObj = new CSS2DObject(dom);
            labelObj.name = 'tooltip';
            labelObj.position.set(intersects[0].point.x,intersects[0].point.y,intersects[0].point.z);
            this.scene.add(labelObj);
          }else{
            tooltipObj.visible = true;
            tooltipObj.position.set(intersects[0].point.x,intersects[0].point.y,intersects[0].point.z);
          }
          this.tooltipText =  item.name;
          // 将几何体添加到场景中
          // intersects[0].object.material = new MeshPhongMaterial()
          // intersects[0].object.material.color.set('red')
        }else{
          this.scene.getObjectByName('tooltip').visible = false;
          this.showTooltip = false;
        }
      },
      // 单个服务器操作
      serverHandle(){
        this.serverObjs.forEach((item)=>{
          if(item.name.split('-').length === 3){
            if(`sever-${this.warn}` == item.name){
              item.material = new MeshPhongMaterial()
              item.material.color.set('red')
            }else{
              item.material.color.set(1,0,0)
            }
          }
        })
      },
      // 过渡
      tween(child,end,type,direction){
        let interval = setInterval(()=>{
          if(type == 'add'){
            child.position[direction] = (child.position[direction] * 10 + 0.1 * 10) / 10
          }else{
            child.position[direction] = (child.position[direction] * 10 - 0.1 * 10) / 10
          }
          if(type == 'add'){
            if(child.position[direction] >= end) {
              clearInterval(interval)
            }
          }else{
            if(child.position[direction] <= end) {
              clearInterval(interval)
            }
          }
        },50)
      },
      // 拆分
      splitHandle(){
        if(this.mesh){
          this.isOpen = !this.isOpen;
          this.mesh.children.forEach((child)=>{
            if (child instanceof Mesh ){
              if(child.name.includes('rdhx')){
                if(child.position.z >= 1){
                  this.tween(child,0,'reduce','z')
                }else if(child.position.z <= 0){
                  this.tween(child,1,'add','z')
                }
              }
              if(child.name.includes('door') || child.name.includes('EMO')|| child.name.includes('minipc')){
                if(child.position.z >= 0){
                  this.tween(child,-1,'reduce','z')
                }else if(child.position.z <= -1){
                  this.tween(child,0,'add','z')
                }
              }
              if(child.name == 'right' || child.name == 'switch1' || child.name == 'switch2'){
                if(child.position.x >= 0){
                  this.tween(child,-1,'reduce','x')
                }else if(child.position.x <= -1){
                  this.tween(child,0,'add','x')
                }
              }
              if(child.name == 'left' || child.name == 'switch3' || child.name == 'switch4'){
                if(child.position.x <= 0){
                  this.tween(child,1,'add','x')
                }else if(child.position.x >= 1){
                  this.tween(child,0,'reduce','x')
                }
              }
            }
          })
        }
      },
      animate() {
        if(this.mesh){
          this.mesh.rotation.y = -10;//添加动画
          // 服务器操作
          this.serverHandle();
          // this.clickObjects.forEach((child)=>{
          //   if(`sever-${this.warn}` == child.name){
          //     child.material = new MeshPhongMaterial()
          //     child.material.color.set('red')
          //   }else{
          //     child.material.color.set(1,0,0)
          //   }
          // })
          this.labelRenderer.render(this.scene,this.camera)
        }
        this.renderer.render(this.scene, this.camera);
        requestAnimationFrame(this.animate);
      },
      // 初始化灯光
      initLight(){
        const ambientLight = new AmbientLight('white');
        this.scene.add(ambientLight);
      },
      // 初始化相机
      initCamera(dom){
        this.camera = new PerspectiveCamera(45,dom.clientWidth / dom.clientHeight,1,1000);
        this.camera.position.z = 6
        this.camera.position.y = 1
        this.scene.add(this.camera);
      },
      // 初始化渲染器
      initRenderer(dom){
        this.renderer = new WebGLRenderer({//渲染器
          antialias: true,
        });
        this.renderer.setClearColor('white');//画布颜色
        this.renderer.setSize(dom.clientWidth,dom.clientHeight)
        document.getElementById( 'container' ).appendChild( this.renderer.domElement );
      },
      // 控制器
      initControl(){
        const orbitControls = new THREE.OrbitControls(this.camera, this.renderer.domElement)
      },
      // 初始化坐标辅助
      initAxis(){
        let axisHelper = new AxisHelper(250);
        this.scene.add(axisHelper);
      },
      // 加载地板
      initGroud(dom){
        const loader = new THREE.TextureLoader;
        loader.load('/bg.jpg',  (texture) =>{
          texture.wrapS = texture.wrapT = THREE.RepeatWrapping; //这里设置x和y超过了图片的像素之后进行的是重复绘制图片操作
          texture.repeat = new THREE.Vector2(50, 50); //设置图片重复绘制的密度这里是5*5
          //设置材质是双面应用该图片材质
          var floorMaterial = new THREE.MeshBasicMaterial({map: texture, side: THREE.DoubleSide});
          //地板使用PlaneGeometry生成平面
          var floorGeometry = new THREE.PlaneGeometry(dom.clientWidth, dom.clientHeight);
          //生成地板的模型
          var floor = new THREE.Mesh(floorGeometry, floorMaterial);
          //设置地板的位置
          floor.position.y = -1;
          floor.rotation.x = Math.PI / 2;
          this.scene.add(floor);//场景加载该地板
        });
      }
    }
  }
</script>
<style lang="less">

  #app{
    display: flex;
    align-items: flex-start;
  }

  .label,.test {
    border-radius: 5px;
    color: white;
    border: 1px solid #fff;
    padding: 10px 5px;
    background: rgba(0, 0, 0, 0.6);
  }

</style>
