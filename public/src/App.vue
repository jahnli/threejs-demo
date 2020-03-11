<template>
  <div id="app">
    <canvas id="container" width="800px" height="600px" ></canvas>
    {{warn}}
  </div>
</template>
<script>
  import {OBJLoader,MTLLoader} from 'three-obj-mtl-loader';
  import * as THREE from 'three';
  import 'three-orbitcontrols';
  import {Vector2,Raycaster,AxisHelper,PointLight,RectAreaLight,PointLightHelper,AmbientLight,WebGLRenderer,PerspectiveCamera,Scene,Mesh,MeshPhongMaterial,SpotLight,SpotLightHelper,DirectionalLight,MeshBasicMaterial,MeshLambertMaterial,DoubleSide} from 'three'
  export default {
    data(){
      return{
        camera: null,
        scene: null,
        renderer: null,
        mesh: null,
        mouse:null,
        raycaster:null,
        clickObjects:[],
        warn:'1-2'
      }
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
        this.initAxis()
        // 加载地板
        this.initGroud(dom);
        this.initControl();
        // 加载模型
        this.initMesh(dom);
      },
      // 初始化模型
      initMesh(dom){
        let mtlLoader = new MTLLoader();
        mtlLoader.load('/hanson.mtl', (materials) =>{
          // 预加载
          materials.preload();
          const loader = new OBJLoader();//在init函数中，创建loader变量，用于导入模型
          loader.setMaterials( materials )
          loader.load('/hanson.obj', (obj)=> {
            //第一个表示模型路径，第二个表示完成导入后的回调函数，一般我们需要在这个回调函数中将导入的模型添加到场景中
            obj.traverse((child) =>{
              if (child instanceof Mesh && child.name.split('-').length == 3 && child.name.split('-')[0] == 'sever') {
                this.clickObjects.push(child)
                child.material.side = DoubleSide;
              }
            });
            dom.addEventListener('mousedown', this.onMouseDown, false);
            obj.position.y = -1
            this.mesh = obj;//储存到全局变量中
            this.scene.add(obj);//将导入的模型添加到场景中
          });
        })
      },
      onMouseDown(event){
        event.preventDefault();
        this.mouse.x = (event.clientX / this.renderer.domElement.clientWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / this.renderer.domElement.clientHeight) * 2 + 1;
        this.raycaster.setFromCamera(this.mouse, this.camera);
        //总结一下，这里必须装网格，mesh，装入组是没有效果的
        //所以我们将所有的盒子的网格放入对象就可以了
        // 需要被监听的对象要存储在clickObjects中。
        const intersects = this.raycaster.intersectObjects(this.clickObjects);
        if(intersects.length > 0) {
          // 在这里填写点击代码
          console.log(intersects[0].object);
          intersects[0].object.material = new MeshPhongMaterial()
          intersects[0].object.material.color.set('red')
        }
      },
      animate() {
        if(this.mesh){
          this.mesh.rotation.y = -10;//添加动画
          this.clickObjects.forEach((child)=>{
            if(`sever-${this.warn}` == child.name){
              child.material = new MeshPhongMaterial()
              child.material.color.set('red')
            }else{
              child.material.color.set(1,0,0)
            }
          })
        }
        requestAnimationFrame(this.animate);
        this.renderer.render(this.scene, this.camera);
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
          canvas:dom, // 画布
          antialias: true,
        });
        this.renderer.setClearColor('white');//画布颜色
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
    },
    mounted() {
      let arr = ['1-1','2-3','3-2','1-4','5-4','3-2','2-1','2-2','2-3','2-4','5-2','8-2','7-1','7-4','6-4','6-2']
      this.init();
      this.animate()
      setInterval(()=>{
        this.warn = arr[Math.floor(Math.random() * 15 + 1)];
      },3000)
    }
  }
</script>
<style lang="less">

  #container {
  }

</style>
