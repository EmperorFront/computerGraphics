<template>
  <div id="space">

  </div>
</template>

<script>
let scene
const GEOLIB = require('geolib');
import * as THREE from 'three'
import { MapControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { Water } from 'three/examples/jsm/objects/Water.js';
import geodata from './edinburgh.js';
import water from './tiananmenWater.js';

import { mergeBufferGeometries }  from 'three/examples/jsm/utils/BufferGeometryUtils.js'

import Stats from 'three/examples/jsm/libs/stats.module.js';

export default {
  name: 'Map',
  props: {
    msg: String
  },
  data() {
    return {
      imgsrc:require('../assets/waternormals.png'),
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      center:[
                116.385279,
                39.9156428
              ],
      // 所有mesh的集合
      geos_building:[],
      // 纹理
      MAT_BUILDING: null,
      MAT_WAY: null,
      MAT_WATER_NORMAL: null,
      MAT_WATER: null,
      stats: null,
      // 点击事件
      raycaster: null,
      // FLAG_ROAD_ANI: true,
      FLAG_ROAD_ANI: false,
      collider_building: [],
      iR: null,
      iR_Line: null,//打个组
      iR_Water: null,
      Animated_Line_Distances: [],
    }
  },
  mounted() {
    console.log(this.imgsrc)
    let _this = this;
    this.init();

    window.addEventListener( 'resize', onWindowResize );
		function onWindowResize() {
      _this.camera.aspect = window.innerWidth / window.innerHeight;
      _this.camera.updateProjectionMatrix();
      _this.renderer.setSize( window.innerWidth, window.innerHeight );
    }
    onWindowResize();

    document.getElementById('space').addEventListener('mousedown',(event)=>{
      let mouse = {
        x: ( event.clientX / window.innerWidth ) * 2 - 1,
	      y: - ( event.clientY / window.innerHeight ) * 2 + 1
      }

      let hitted = _this.Fire(mouse)
      console.log(hitted)
    })
  },
  methods: {
    init() {
      let cont = document.getElementById('space');

      this.stats = new Stats()
      cont.appendChild(this.stats.domElement)

      //init scent
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x222222)
      //init camera
      this.camera = new THREE.PerspectiveCamera(25, window.clientWidth/window.clientHeight, 1, 100);
      this.camera.position.set(8,4,0)

      //init raycaster 点击事件
      this.raycaster = new THREE.Raycaster();

      //init group
      this.iR = new THREE.Group();
      this.iR.name = 'interactive root'
      this.iR_Line = new THREE.Group();
      this.iR_Line.name = ''
      this.iR_Water = new THREE.Group();
      this.iR_Water.name = ''
      this.iR_Water.position.set(-0.24,0.37,0);
      // 在这里旋转会导致water纹理绘制到模型上出错，因此挪到下边去弄
      // this.iR_Water.rotateY(-0.08); 
      this.$nextTick(()=>{
        scene.add(this.iR)
        scene.add(this.iR_Line)
        scene.add(this.iR_Water);
      });

      //init light
      let light0 = new THREE.AmbientLight(0xfafafa,0.25)
      let light1 = new THREE.PointLight(0xfafafa,0.4)
      light1.position.set(200,90,40)
      let light2 = new THREE.PointLight(0xfafafa,0.45)
      light2.position.set(200,90,-40)

      scene.add( light0 );
      scene.add( light1 );
      scene.add( light2 );
      
      let gridHelper = new THREE.GridHelper(60, 160, new THREE.Color(0x555555), new THREE.Color(0x333333))
      scene.add(gridHelper);

      //test
      // const geometry = new THREE.BoxGeometry( 1, 1, 1 );
      // const material = new THREE.MeshBasicMaterial( {color: 0x00ff00} );
      // const cube = new THREE.Mesh( geometry, material );
      // scene.add( cube );

      //init renderer
      this.renderer = new THREE.WebGLRenderer({antialias:true})
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.setSize(window.innerWidth,window.innerHeight)

      cont.appendChild(this.renderer.domElement);

      //init cnotrols
      this.controls = new MapControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true; // an animation loop is required when either damping or auto-rotation are enabled
      this.controls.dampingFactor = 0.25;
      this.controls.screenSpacePanning = false;
      // this.controls.minDistance = 100;
      this.controls.maxDistance = 800;
      // this.controls.maxPolarAngle = Math.PI / 2;
      this.controls.update();

      this.MAT_BUILDING = new THREE.MeshPhongMaterial()
      this.MAT_WAY = new THREE.LineBasicMaterial({color:0x456390})
      //更新渲染
      this.Update()
      this.LoadBuilding(geodata.geodata)
      this.loadWaters(water.water)
    },
    Update() {
      requestAnimationFrame(this.Update)      
      this.renderer.render(scene, this.camera)
      this.controls.update();
      this.UpdateWater();
      this.stats.update();
      this.UpdateAnimatedLine();
    },

    LoadBuilding(data,datas){

      let features = data.features

      for(let i = 0; i < features.length; i++){
        let fel = features[i]
        if (!fel['properties']) return

        //几何体信息
        let info = fel.properties

        if (info['building']){
          this.addBuilding(
            fel.geometry.coordinates, 
            info, 
            info['building:levels']);
        } else if(
            fel.geometry.type == "LineString" 
            && 
            info['highway'] != "pedestrian" &&
            info['highway'] != "footway" &&//人行道
            info['highway'] != "path" //小路径
            ){
            //画路
            this.addRoad(fel.geometry.coordinates, info)
        }
      }
      let _this = this
      this.$nextTick(()=>{
        let mergeGeometry = mergeBufferGeometries(this.geos_building);
        let mesh = new THREE.Mesh(mergeGeometry, _this.MAT_BUILDING)
        mesh.position.set(-0.23,0.5,0)
        mesh.rotateY(-0.08); 
        scene.add(mesh);

        // this.loadWaters(water.water)
      });
    },
    loadWaters(water){      
      let _this = this;
      this.MAT_WATER_NORMAL = new THREE.TextureLoader().load(_this.imgsrc, (texture)=>{
        console.log('afterLoader',texture)
        texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
      })
      this.MAT_WATER = {
        textureWidth: 512,
        textureHeight: 512,
        waterNormals: this.MAT_WATER_NORMAL,
        sunDirection: new THREE.Vector3(),
        sunColor: 0xffffff,
        waterColor: 0xA6C8FA,
        distortionScale: 3.7,
        fog: false
      }
      let features = water.features;
      for (let i = 0; i < features.length; i++) {
        let fel = features[i]
        if(fel.properties['natural'] == 'water' && fel.geometry.type == 'Polygon') {
          this.addWater(fel.geometry.coordinates, fel.properties);
        }
      }
    },
    addWater(data, info){
      let shape, geometry;
      let holes = [];
      for(let i = 0; i< data.length; i++) {
        let el = data[i];
        if(i == 0) {
          shape = this.genShape(el, this.center)
        } else {
          holes.push(this.genShape(el, this.center))
        }
      }
      for(let i = 0; i< holes.length; i++){
        shape.holes.push(holes[i])
      }
      geometry = this.genGeometry(shape,{	
        curveSegmetn:1,
        depth: 0.1,
        bevelEnabled: false
      })
      geometry.rotateX(Math.PI/2);      
      geometry.rotateZ(Math.PI);
      geometry.rotateY(-0.08); 

      let water = new Water(geometry, this.MAT_WATER);

      this.iR_Water.add(water)
    },
    addBuilding(data, info,height=1) {
      height = height ? height : 1
      for(let i = 0; i <data.length; i++) {
        let el =data[i]
        let shape=this.genShape(el,this.center);
        let geometry = this.genGeometry(shape,{	
            curveSegmetn:1,
            depth: 0.1*height,
            bevelEnabled: false
          })
          geometry.rotateX(Math.PI/2);      
          geometry.rotateZ(Math.PI);

          //把所有mesh组合为一个
          this.geos_building.push(geometry)

          let helper = this.genHelper(geometry)

          if(helper){
            helper.name = info['name'] ? info['name']: 'Building' 
            helper.info = info
            this.collider_building.push(helper)
          }
            
          // let mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
          // scene.add(mesh);
        }
    },
    addRoad(data, info){
      let points = []
      for(let i =0; i<data.length; i++) {
        if(!data[0][1])return
        let el = data[i]
        if(!el[0] || !el[1]) return
        let elp = [el[0],el[1]]
        elp = this.GPSRelativePosition([elp[0], elp[1]], this.center)
        points.push(new THREE.Vector3(elp[0], 0.5, elp[1]))
      }
      let geometry = new THREE.BufferGeometry().setFromPoints(points)
      let line = new THREE.Line(geometry,this.MAT_WAY)
      line.computeLineDistances()

      scene.add(line)
      // line.position.y = 0.5
      if(this.FLAG_ROAD_ANI){
        //启动动画
        let lineLength = geometry.attributes.lineDistance.array[
            geometry.attributes.lineDistance.count - 1
          ]
        if(lineLength > 0.8){
          let aniLine = this.addAnimateLine(geometry, lineLength);
          this.iR_Line.add(aniLine)
        }
      }
    },
    addAnimateLine(geometry, length) {
      let animatedLine = new THREE.Line(geometry, new THREE.LineDashedMaterial({color:0x00FFFF}))
      animatedLine.material.dashSize = 0;
      animatedLine.material.gapSize = 1000;
      animatedLine.position.y = 0.5;
      animatedLine.material.transparent = true;
      
      this.Animated_Line_Distances.push(length)

      return animatedLine
    },
    UpdateAnimatedLine(){
      if(this.iR_Line.children.length <= 0) return

      for(let i =0; i < this.iR_Line.children.length; i++){
        let line = this.iR_Line.children[i]
        let dash = parseInt(line.material.dashSize)
        let length = parseInt(this.Animated_Line_Distances[i])
        if(dash > length){
          //recover
          line.material.dashSize = 0;
          line.material.opacity = 1;
        }else {
          //animate
          line.material.dashSize += 0.028
          line.material.opacity = line.material.opacity > 0 ? line.material.opacity - 0.001: 0;
        }
      }
    },
    UpdateWater(){
      for (let i = 0; i< this.iR_Water.children.length; i++){
        this.iR_Water.children[i].material.uniforms['time'].value += 1.0/700
      }
    },
    genHelper(geometry){
      if(geometry.boundingBox){
          geometry.computeBoundingBox()
      }
      let box3 = geometry.boundingBox;
      if(!isFinite(box3.max.x)){
        return false
      }
      let helper = new THREE.Box3Helper( box3, 0xffff00 )
      helper.updateMatrixWorld()
      return helper
    },
    genShape(points,center){
      let shape = new THREE.Shape()
      for(let i=0; i<points.length; i++){
        let elp = points[i];
        elp = this.GPSRelativePosition(elp,center)

        if(i == 0){
          shape.moveTo(elp[0],elp[1])
        } else {
          shape.lineTo(elp[0],elp[1])
        }
      }
      return shape
    },
    Fire(mouse){
      this.raycaster.setFromCamera(mouse, this.camera);
      let intersects = this.raycaster.intersectObjects(this.collider_building, true)
      if(intersects.length>0){
        return intersects[0].object
      } else {
        return false
      }
    },
    genGeometry (shape, config){
      let geometry = new THREE.ExtrudeBufferGeometry(shape, config);
      geometry.computeBoundingBox();
      return geometry
    },
    GPSRelativePosition(obj,center){
      let dis = GEOLIB.getDistance(obj,center);
      let bearing = GEOLIB.getRhumbLineBearing(obj,center);
      let x = center[0] + (dis*Math.cos(bearing*Math.PI/180))
      let y = center[1]+(dis*Math.sin(bearing*Math.PI/180))
      return [-x/100,y/100]
    },
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#space {
  width: 100%;
  height: 100%;
}
</style>
