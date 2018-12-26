<template>
  <div class="outer-bg">
    <div
      id="graphContainer"
      class="process-tree-container"
    />
    <div class="control-wrapper">
      <button
        style="margin-right:20px"
        @click="setCameraPosition1"
      >俯瞰图</button>
      <button @click="setCameraPosition2">平视图</button>
    </div>
  </div>
</template>

<script>
// import { processTree } from "@/api/mergeDetail";
import tree from "./testTree";
// import { mapState } from "vuex";
import * as THREE from "three";
import $ from "jquery";
let globalScale = 1;
let pointLights = [];
let minDistance = 6;
let maxArrangeLevel = 0;
let animationFrames = {};
let id = 0;
let fov = 1;
let near = 0.001;
let far = 1000000;
let links = [];
let brushing = false;
let brushStartPosition = [];
let dragging = false;
let draggingCanvas = false;
let draggingCanvasStartPosition = [];
let globalControlNode = {};
let globalDragX = 0;
let globalDragY = 0;
let globalEx = 0;
let globalEy = 0;
let globalMemoryEx = 0;
let globalMemoryEy = 0;
let curSceneAnimation = null;

function getTop(e) {
  var offset = e.offsetTop;
  if (e.offsetParent != null) offset += getTop(e.offsetParent);
  return offset;
}

function getLeft(e) {
  var offset = e.offsetLeft;
  if (e.offsetParent != null) offset += getLeft(e.offsetParent);
  return offset;
}

export default {
  data() {
    return {
      renderer: null,
      camera: null,
      scene: null,
      mesh: null,
      miserables: {},
      nodesArr: []
    };
  },

  mounted() {
    window.container = document.getElementById("graphContainer");
    window.width = container.offsetWidth;
    window.height = container.offsetHeight;
    setTimeout(() => {
      this.init();
    }, 200);
  },
  // destroyed() {},
  methods: {
    async initData() {
      await (this.miserables = tree);
    },
    async init() {
      await this.initData();
      await this.nodesArr.push(this.miserables);
      await this.pushIntoNodesArr(this.miserables);
      // console.log(this.miserables)
      // console.log("miserables", this.miserables);
      await this.initMesh();
      await this.createBackground();
      this.miserables.x = 0;
      this.miserables.y = 0;
      this.miserables.z = 0;
      await this.createNode(this.miserables);
      await this.setPosition(this.miserables);
      await this.createLinks(this.miserables);
      await this.initLight();
      await this.animate();

      const canvas = document.querySelector("canvas");
      canvas.addEventListener("mousewheel", this.mousewheel, false);
      canvas.addEventListener("DOMMouseScroll", this.mousewheel, false);
      canvas.addEventListener("mousedown", this.mousedown, false);
      canvas.addEventListener("mouseup", this.mouseup, false);
      canvas.addEventListener("mousemove", this.mousemove, false);
    },

    animate() {
      let len = Object.getOwnPropertyNames(animationFrames).length;
      animationFrames["s" + len] = timestamp => {
        this.render();
        if (animationFrames["s" + len]) {
          window.requestAnimationFrame(animationFrames["s" + len]);
        }
      };
      window.requestAnimationFrame(animationFrames["s" + len]);
    },

    setCameraPosition1() {
      this.camera.position.set(-1400, 1900, 2400);
    },

    setCameraPosition2() {
      this.camera.position.set(-2705.9752444222386, 4754, 11.574825009807226);
    },

    pushIntoNodesArr(obj) {
      for (var i = 0; i < obj.children.length; i++) {
        if (this.nodesArr.indexOf(obj.children[i]) < 0) {
          this.nodesArr.push(obj.children[i]);
        }
        let cChildren = obj.children[i].children || [];
        // for(var i=0;i<obj.children.length;i++){
        if (cChildren.length > 0) {
          this.pushIntoNodesArr(obj.children[i]);
        }
      }
    },

    initMesh() {
      let container = window.container;
      let width = window.width;
      let height = window.height;

      this.scene = new THREE.Scene(); // 场景
      this.camera = new THREE.PerspectiveCamera(fov, width / height, near, far);
      this.camera.nextZoom = this.camera.zoom = 1;
      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        precision: "lowp",
        alpha: true
      });
      this.renderer.setSize(width, height);
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setClearColor("#071a28", 1);
      container.appendChild(this.renderer.domElement);
      this.camera.position.set(-1400, 1900, 2400);

      this.textureLoader = new THREE.TextureLoader();
      this.particleGraphic = this.textureLoader.load("/img/normal.png");
      this.particleGraphic2 = this.textureLoader.load("/img/abnormal.png");
      this.auraGraphic = this.textureLoader.load("/img/normal-selected.png");
      this.auraGraphic2 = this.textureLoader.load("/img/abnormal-selected.png");
      // console.log('this.particleGraphic: ', this.particleGraphic)
      // console.log('this.particleGraphic2: ', this.particleGraphic2)
      // console.log('this.auraGraphic: ', this.auraGraphic)
      // console.log('this.auraGraphic2: ', this.auraGraphic2)
    },

    createBackground() {
      this.material = new THREE.MeshPhongMaterial({
        color: "#2C3855",
        dithering: false
      });
      this.geometry = new THREE.PlaneBufferGeometry(2000, 2000);
      this.mesh = new THREE.Mesh(this.geometry, this.material);
      this.mesh.position.set(0, 1, 0);
      this.mesh.rotation.x = -Math.PI * 0.5;
      this.mesh.receiveShadow = true;
      this.scene.add(this.mesh);
    },

    createNodeText(node) {
      let textMaterial = new THREE.SpriteMaterial({
        color: "#fff",
        map: new THREE.CanvasTexture(this.createSpriteText(node.name)),
        blending: THREE.AdditiveBlending,
        transparent: false,
        depthTest: false,
        opacity: 1
      });
      node.textGragh = new THREE.Sprite(textMaterial);
      node.textGragh.scale.x = node.textGragh.scale.y = 4;
      node.textGragh.position.set(0, -1.5, 0);
      node.circle.add(node.textGragh);
    },

    createLinkText(link) {
      let textMaterial = new THREE.SpriteMaterial({
        map: new THREE.CanvasTexture(
          this.createSpriteText2(link.target.relationWithParent)
        ),
        blending: THREE.AdditiveBlending,
        transparent: false,
        depthTest: false,
        opacity: 1
      });
      link.textGragh = new THREE.Sprite(textMaterial);
      link.textGragh.scale.x = link.textGragh.scale.y = 5;

      link.textGragh.position.set(
        link.target.x,
        link.target.y,
        link.target.z - minDistance / 3
      );

      this.scene.add(link.textGragh);
    },

    getSquareShape() {
      var squareShape = new THREE.Shape();
      squareShape.moveTo(0, 0);
      squareShape.lineTo(0, 5);
      squareShape.lineTo(5, 5);
      squareShape.lineTo(5, 0);
      squareShape.lineTo(0, 0);
      return squareShape;
    },

    createNode(node) {
      let container = window.container;
      let icon;
      let auraMaterial = new THREE.SpriteMaterial({
        map: this.auraGraphic,
        color: "#fff",
        transparent: true,
        opacity: 0,
        alphaTest: 0.1,
        depthTest: false
      });

      let circleMaterial = new THREE.SpriteMaterial({
        map: this.particleGraphic,
        alphaTest: 0.1,
        color: "#fff",
        transparent: true,
        opacity: 1,
        depthTest: false
      });

      if (Math.random() > 0.5) {
        circleMaterial.map = this.particleGraphic2;
        auraMaterial.map = this.auraGraphic2;
      }

      node.aura = new THREE.Sprite(auraMaterial);
      // node.aura.frustumCulled = true;
      node.aura.modelType = "aura";
      // node.aura.renderOrder = i+data.nodes.length;
      // node.aura.scale.x = node.aura.scale.y = Math.pow(Math.ceil(Math.random()*180), Math.pow(Math.random(), 1));
      node.aura.scale.x = node.aura.scale.y = 1;

      node.circle = new THREE.Sprite(circleMaterial);
      node.circle.modelType = "circle";
      //  node.circle.renderOrder = i;
      // nodeRadius = 2
      node.circle.scale.x = node.circle.scale.y = node.circle.scale.z = 2;
      node.circle.add(node.aura);
      node.icons = [];
      //添加小图标
      if (Math.random() > 0.5) {
        // var iconMaterial = new THREE.SpriteMaterial({map:correspondGraph, alphaTest :0.1 ,transparent: false, opacity:1,depthTest: false});
        // var icon = new THREE.Sprite(iconMaterial);
        icon = $(
          '<img style="position:absolute;z-index:1000;" src="/img/correspond.png"></img>'
        );
        $(container).append(icon);
        node.icons.push(icon);
      }
      if (Math.random() > 0.5) {
        // var iconMaterial = new THREE.SpriteMaterial({map:fileGraph, alphaTest :0.1 ,transparent: false, opacity:1,depthTest: false});
        // var icon = new THREE.Sprite(iconMaterial);
        icon = $(
          '<img style="position:absolute;z-index:1000;" src="/img/file.png"></img>'
        );
        $(container).append(icon);
        node.icons.push(icon);
      }
      if (Math.random() > 0.5) {
        // var iconMaterial = new THREE.SpriteMaterial({map:injectionGraph, alphaTest :0.1 ,transparent: false, opacity:1,depthTest: false});
        // var icon = new THREE.Sprite(iconMaterial);
        icon = $(
          '<img style="position:absolute;z-index:1000;" src="/img/injection.png"></img>'
        );
        $(container).append(icon);
        node.icons.push(icon);
      }
      if (Math.random() > 0.5) {
        // var iconMaterial = new THREE.SpriteMaterial({map:loadGraph, alphaTest :0.1 ,transparent: false, opacity:1,depthTest: false});
        // var icon = new THREE.Sprite(iconMaterial);
        icon = $(
          '<img style="position:absolute;z-index:1000;" src="/img/load.png"></img>'
        );
        $(container).append(icon);
        node.icons.push(icon);
      }
      if (Math.random() > 0.5) {
        // var iconMaterial = new THREE.SpriteMaterial({map:regeditGraph, alphaTest :0.1 ,transparent: false, opacity:1,depthTest: false});
        // var icon = new THREE.Sprite(iconMaterial);
        icon = $(
          '<img style="position:absolute;z-index:1000;" src="/img/regedit.png"></img>'
        );
        $(container).append(icon);
        node.icons.push(icon);
      }

      this.createNodeText(node);
      node.circle.position.set(node.x, node.y, node.z);

      this.scene.add(node.circle);
    },

    setPosition(obj) {
      let maxZ = 0;
      let len = obj.children || [];

      for (var i = 0; i < len.length; i++) {
        len[i].x =
          obj.x - ((len.length - 1) * minDistance) / 2 + i * minDistance;
        len[i].arrangeLevel = obj.arrangeLevel + 1;

        // if (!gradedNodes['s' + len[i].arrangeLevel]) {
        //   gradedNodes['s' + len[i].arrangeLevel] = []
        //   maxArrangeLevel = len[i].arrangeLevel
        // }
        // if (
        //   gradedNodes['s' + len[i].arrangeLevel].indexOf(
        //     len[i]
        //   ) < 0
        // ) {
        //   gradedNodes['s' + len[i].arrangeLevel].push(len[i])
        // }

        len[i].y = obj.y;

        len[i].z = obj.z + minDistance;
        if (len[i].z > maxZ) {
          maxZ = len[i].z;
        }
        // updateNodePosition(obj);
        if (!len[i].circle) {
          this.createNode(len[i]);
        }

        let cChildren = len[i].children || [];
        if (cChildren.length > 0) {
          this.setPosition(len[i]);
        }
      }
    },

    createLink(source, target) {
      let linkMaterial = new THREE.LineBasicMaterial({
        color: "rgb(78, 97, 152)",
        transparent: false,
        opacity: 0.1,
        alphaTest: 0.1,
        depthTest: false
      });
      let link = {
        source,
        target
      };
      link.material = linkMaterial;
      link.geometry = new THREE.Geometry();
      link.line = new THREE.Line(link.geometry, link.material);

      link.line.geometry.verticesNeedUpdate = true;
      link.line.geometry.vertices[0] = new THREE.Vector3(
        source.x,
        source.y,
        source.z
      );
      link.line.geometry.vertices[1] = new THREE.Vector3(
        source.x,
        (source.y + target.y) / 2,
        (source.z + target.z) / 2
      );
      link.line.geometry.vertices[2] = new THREE.Vector3(
        target.x,
        (source.y + target.y) / 2,
        (source.z + target.z) / 2
      );
      link.line.geometry.vertices[3] = new THREE.Vector3(
        target.x,
        target.y,
        target.z
      );
      link.line.geometry.computeBoundingSphere();
      link.line.frustumCulled = false;

      links.push(link);
      this.scene.add(link.line);
      this.createLinkText(link);
    },

    createLinks(obj) {
      for (var i = 0; i < obj.children.length; i++) {
        var exist = false;
        for (var j = 0; j < links.length; j++) {
          if ((links[j].source == obj, links[j].target == obj.children[i])) {
            exist = true;
            break;
          }
        }
        if (!exist) {
          this.createLink(obj, obj.children[i]);
        }

        let cChildren = obj.children[i].children || [];
        if (cChildren.length > 0) {
          this.createLinks(obj.children[i]);
        }
      }
    },

    createSpriteText(text) {
      //先用画布将文字画出
      const borderThickness = 2;
      let canvas = document.createElement("canvas");
      canvas.width = 512;
      canvas.height = 512;
      let ctx = canvas.getContext("2d");
      ctx.font = "Bold 25px Times New Roman";
      /* 获取文字的大小数据，高度取决于文字的大小 */
      let metrics = ctx.measureText(" " + text);
      let textWidth = metrics.width;

      ctx.fillStyle = "#000";
      ctx.fillStyle = "rgb(170, 190, 230)";

      ctx.textAlign = "center";
      ctx.fillText(text, 256, 25 + borderThickness, 512);
      return canvas;
    },

    createSpriteText2(text) {
      //先用画布将文字画出
      let canvas = document.createElement("canvas");
      canvas.width = 512;
      canvas.height = 512;
      let ctx = canvas.getContext("2d");
      ctx.fillStyle = "rgb(170, 190, 230)";
      ctx.font = "Bold 30px Times New Roman";
      ctx.textAlign = "center";
      ctx.fillText(text, 256, 300, 512);

      return canvas;
    },

    initLight() {
      this.scene.add(new THREE.AmbientLight(0x404040));

      var dirlight = new THREE.DirectionalLight(0xffffff);

      dirlight.position.set(1, 1, 1);
      this.scene.add(dirlight);

      var pointLight = new THREE.PointLight(0xeeeeff, 0.1);
      pointLight.position.set(0, 6, 0);
      pointLight.intensity = 0;
      pointLight.obj = { x: 0, y: 0, z: 0 };
      this.scene.add(pointLight);
      pointLights.push(pointLight);
    },

    render() {
      const container = document.getElementById("graphContainer");
      window.width = container.offsetWidth;
      window.height = container.offsetHeight;

      if (curSceneAnimation) {
        if ((curSceneAnimation = "horizontalRotate")) {
          //圆心角
          var timestamp = new Date().getTime();
          var radius = width * 2;
          var curXAngle = -(((timestamp / 1000) % 360) * Math.PI) / 30;
          var curY = Math.sin((((timestamp / 60) % 360) * Math.PI) / 180) + 1;

          var curX = -radius * Math.sin(curXAngle);
          var curZ = radius * Math.cos(curXAngle);
          this.camera.position.x = curX;
          this.camera.position.z = curZ;
          this.camera.position.y = (radius * curY) / 2;
        }
      }

      pointLights[0].position.x = pointLights[0].obj.x;
      pointLights[0].position.z = pointLights[0].obj.z;

      var angle = Math.atan(this.camera.position.z / this.camera.position.x);

      if (globalEx != 0 && globalEx != globalMemoryEx && globalMemoryEx != 0) {
        var diffX = globalEx - globalMemoryEx;
        // console.log(diffX)
        // console.log(this.nodesArr)
        this.nodesArr.forEach(node => {
          node.x +=
            (diffX / 10 / this.camera.zoom) *
            (this.camera.position.x < 0 ? -1 : 1) *
            Math.sin(angle);
          node.z +=
            (diffX / 10 / this.camera.zoom) *
            (this.camera.position.x < 0 ? 1 : -1) *
            Math.cos(angle);
        });
      }
      if (globalEy != 0 && globalEy != globalMemoryEy && globalMemoryEy != 0) {
        var diffY = globalEy - globalMemoryEy;
        this.nodesArr.forEach(node => {
          node.z +=
            ((diffY / 10 / this.camera.zoom) *
              (this.camera.position.x < 0 ? -1 : 1) *
              Math.sin(angle) *
              width) /
            height;
          node.x +=
            ((diffY / 10 / this.camera.zoom) *
              (this.camera.position.x < 0 ? -1 : 1) *
              Math.cos(angle) *
              width) /
            height;
        });
      }
      globalMemoryEy = globalEy;
      globalMemoryEx = globalEx;

      if (this.camera.zoom != this.camera.nextZoom) {
        if (
          Math.abs(
            Math.abs(this.camera.nextZoom) - Math.abs(this.camera.zoom)
          ) < 0.001
        ) {
          this.camera.zoom = this.camera.nextZoom;
        } else {
          this.camera.zoom += (this.camera.nextZoom - this.camera.zoom) / 9;
        }
        this.camera.updateProjectionMatrix();
      }
      this.nodesArr.forEach((node, index) => {
        const { x, y, z, aura, circle, textGragh } = node;
        circle.position.set(x, y, z);

        let v = new THREE.Vector3(
          node.circle.position.x,
          node.circle.position.y,
          node.circle.position.z
        );
        const xy = v.project(this.camera);
        let finalXY = {
          x: ((xy.x + 1) * width) / 2,
          y: ((1 - xy.y) * height) / 2
        };

        for (var i = 0; i < node.icons.length; i++) {
          //  node.icons[i].position.set(Math.cos(camera.rotation.y) * (i * 0.15 - 0.5), 0.5, -Math.sin(camera.rotation.y) * (i * 0.15 - 0.5));
          //node.icons[i].position.set(Math.cos(camera.rotation.y) * (i * 0.15 - 0.5), 0.5, -Math.sin(camera.rotation.y) * (i * 0.15 - 0.5));
          node.icons[i].css({
            left: finalXY.x + (-16 + 6 * i) * this.camera.zoom,
            top: finalXY.y + -11 * this.camera.zoom,
            transform: "scale(" + this.camera.zoom / 3 + ")"
          });
        }
      });

      var link = links[0];
      var v1 = new THREE.Vector3(link.source.x, link.source.y, link.source.z);
      var xy1 = v1.project(this.camera);
      var finalXY1 = {
        x: ((xy1.x + 1) * width) / 2,
        y: ((1 - xy1.y) * height) / 2
      };

      var v2 = new THREE.Vector3(
        link.source.x,
        (link.source.y + link.target.y) / 2,
        (link.source.z + link.target.z) / 2
      );
      var xy2 = v2.project(this.camera);
      var finalXY2 = {
        x: ((xy2.x + 1) * width) / 2,
        y: ((1 - xy2.y) * height) / 2
      };

      var slope = (finalXY2.y - finalXY1.y) / (finalXY2.x - finalXY1.x);

      links.forEach(link => {
        link.line.geometry.verticesNeedUpdate = true;
        link.line.geometry.vertices[0] = new THREE.Vector3(
          link.source.x,
          link.source.y,
          link.source.z
        );
        link.line.geometry.vertices[1] = new THREE.Vector3(
          link.source.x,
          (link.source.y + link.target.y) / 2,
          (link.source.z + link.target.z) / 2
        );
        link.line.geometry.vertices[2] = new THREE.Vector3(
          link.target.x,
          (link.source.y + link.target.y) / 2,
          (link.source.z + link.target.z) / 2
        );
        link.line.geometry.vertices[3] = new THREE.Vector3(
          link.target.x,
          link.target.y,
          link.target.z
        );
        link.line.geometry.computeBoundingSphere();
        link.textGragh.position.set(
          link.target.x,
          link.target.y,
          link.target.z - minDistance / 3
        );
        link.textGragh.material.rotation = -Math.atan(slope);
      });

      this.camera.lookAt(this.scene.position);
      this.renderer.render(this.scene, this.camera);
    },

    mouseup() {
      //console.log('up');
      globalEx = 0;
      globalEy = 0;
      draggingCanvas = false;
      dragging = false;
      brushing = false;
    },

    mousedown() {
      let mousedownOnNode = false;
      let container = window.container;
      let width = window.width;
      let height = window.height;

      mousedownOnNode = false;
      globalControlNode = null;
      //for (var i = 1; i < pointLights.length; i++) {
      //  this.scene.remove(pointLights[i]);
      //  pointLights.splice(i, 1);
      //  i--;
      //}
      pointLights[0].intensity = 0;
      var distance = 25 * this.camera.zoom;
      var nearestNode = null;
      if (event.buttons == 1) {
        this.nodesArr.forEach((node, index) => {
          const { x, y, z, aura, circle, textGragh } = node;
          circle.position.set(x, y, z);
          node.circle.material.opacity = 1;
          node.aura.material.opacity = 0;
          const v = new THREE.Vector3(
            node.circle.position.x,
            node.circle.position.y,
            node.circle.position.z
          );
          const xy = v.project(this.camera);
          //console.log(xy)
          let finalXY = {
            x:
              ((xy.x + 1) * width) / 2 +
              getLeft(container) +
              $(document).scrollLeft(),
            y:
              ((1 - xy.y) * height) / 2 +
              getTop(container) +
              $(document).scrollTop()
          };
          //console.log(finalXY)
          var dis = Math.pow(
            Math.pow(event.clientX - finalXY.x, 2) +
              Math.pow(event.clientY - finalXY.y, 2),
            0.5
          );
          if (dis < distance) {
            distance = dis;
            nearestNode = node;
          }
        });
        if (nearestNode) {
          mousedownOnNode = true;
          // console.log(nearestNode)
          nearestNode.circle.material.opacity = 0;
          nearestNode.aura.material.opacity = 1;
          globalControlNode = nearestNode;

          console.log("点击了一个节点", nearestNode);

          pointLights[0].obj = globalControlNode;
          pointLights[0].intensity = 1.4;
        }

        if (!mousedownOnNode) {
          draggingCanvas = true;
          draggingCanvasStartPosition = [event.clientX, event.clientY];
          this.camera.startDraggingPosition = this.camera.position;
        }
      }
    },

    mousewheel(e) {
      var di;
      // e.stopPropagation()
      if (e.wheelDelta) {
        if (e.wheelDelta > 0) {
          di = 1;
          globalScale *= globalScale < far ? 1.15 : 1;
        }
        if (e.wheelDelta < 0) {
          di = -1;
          globalScale /= near < globalScale ? 1.15 : 1;
        }
      } else if (e.detail) {
        if (e.detail > 0) {
          di = 1;
          globalScale *= 1.15;
        }
        if (e.detail < 0) {
          di = -1;
          globalScale /= 1.15;
        }
      }

      this.camera.nextZoom = globalScale;
    },

    mousemove(event) {
      // console.log(draggingCanvas);

      if (brushing) {
        let Sx = event.clientX; //鼠标单击位置横坐标
        let Sy = event.clientY; //鼠标单击位置纵坐标
        let lx = Sx > brushStartPosition[0] ? brushStartPosition[0] : Sx;
        let rx = brushStartPosition[0] > Sx ? brushStartPosition[0] : Sx;
        let ty = Sy > brushStartPosition[1] ? brushStartPosition[1] : Sy;
        let by = brushStartPosition[1] > Sy ? brushStartPosition[1] : Sy;
      } else if (dragging) {
      } else if (draggingCanvas) {
        globalDragX = event.clientX - draggingCanvasStartPosition[0];
        globalDragY = event.clientY - draggingCanvasStartPosition[1];

        globalEx = event.clientX;
        globalEy = event.clientY;
      }
    }
  }
};
</script>

<style scoped>
/* 树布局相关 */
.outer-bg {
  background-color: #2C3855;
  height: 740px;
  position: relative;
  width: 100%;
  overflow: hidden;
}
.process-tree-container {
  background-color: #2C3855;
  width: 100%;
  height: 100%;
}

.control-wrapper {
  position: absolute;
  right: 20px;
  top: 20px;
}

</style>
