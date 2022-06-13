<template>
  <div id="space">

  </div>
</template>

<script>
let scene
const GEOLIB = require('geolib');
import * as THREE from 'three'
import { MapControls } from 'three/examples/jsm/controls/OrbitControls.js';
import geodata from './edinburgh.js'

export default {
  name: 'Map',
  props: {
    msg: String
  },
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      center:[
                116.385279,
                39.9156428
              ],
      MAT_BUILDING: null
    }
  },
  mounted() {
    let _this = this;
    this.init();

    window.addEventListener( 'resize', onWindowResize );
		function onWindowResize() {
      _this.camera.aspect = window.innerWidth / window.innerHeight;
      _this.camera.updateProjectionMatrix();
      _this.renderer.setSize( window.innerWidth, window.innerHeight );
    }
    onWindowResize();
  },
  methods: {
    init() {
      let cont = document.getElementById('space');
      //init scent
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x222222)
      //init camera
      this.camera = new THREE.PerspectiveCamera(25, window.clientWidth/window.clientHeight, 1, 100);
      this.camera.position.set(8,4,0)
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
      //更新渲染
      this.Update()
      this.LoadBuilding(geodata.geodata)
    },
    Update() {
      requestAnimationFrame(this.Update)      
      this.renderer.render(scene, this.camera)
      this.controls.update()
    },

    LoadBuilding(data){

      let features = data.features
      
      for(let i = 0; i < features.length; i++){
        let fel = features[i]
        if (!fel['properties']) return
        if (fel.properties['building']){
          
          this.addBuilding(fel.geometry.coordinates, fel.properties, fel.properties['building:levels']);
        }
      }
    },
    addBuilding(data, info,height=1) {;
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

        const material2 = new THREE.MeshBasicMaterial( {color: 0x00ff00} );
       let mesh 
        if(i == 1){
          mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
        }else{
          mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
          }
        

        scene.add(mesh)
        ;
      }
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
