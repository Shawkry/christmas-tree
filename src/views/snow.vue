<template>
  <div id='container' style="display: inline"></div>
</template>

<script>
import * as THREE from 'three';

const scene = new THREE.Scene();
export default {
  name: "PageThird",
  mounted() {
    this.draw();
  },
  methods: {
    draw() {
      console.log("this is main js for piece - snowfall in project threejs");

//https://codepen.io/shubniggurath/pen/WgJZJo

      let vertexShader = `
    void main() {
        gl_Position = vec4( position, 1.0 );
    }
`;
      let fragmentShader = `
    uniform vec2 u_resolution;
    uniform vec2 u_mouse;
    uniform float u_time;
    uniform sampler2D u_noise;

    #define PI 3.141592653589793
    #define TAU 6.

    const float multiplier = 25.5;

    const float zoomSpeed = 10.;
    const int layers = 10;

    const int octaves = 6;

    vec2 hash2(vec2 p)
    {
        vec2 o = texture2D( u_noise, (p+0.5)/256.0, -100.0 ).xy;
        return o;
    }

    mat2 rotate2d(float _angle){
        return mat2(cos(_angle),sin(_angle),
                    -sin(_angle),cos(_angle));
    }

    vec3 hsb2rgb( in vec3 c ){
        vec3 rgb = clamp(abs(mod(c.x*6.0+vec3(0.0,4.0,2.0),
                                6.0)-3.0)-1.0,
                        0.0,
                        1.0 );
        rgb = rgb*rgb*(3.0-2.0*rgb);
        return c.z * mix( vec3(1.0), rgb, c.y);
    }

    //依据实际的坐标，获取对应位置的noise贴图的Red值
    float hash(vec2 p)
    {
        //这边的256是否是因为noise图片的尺寸是256*256
        float o = texture2D( u_noise, (p+0.5)/256.0, -100.0 ).x;//这里疑似多了一个-100.0的参数？
        return o;
    }

    //针对一组浮点二维向量坐标，通过插值的方法，求出其对应的色值
    float noise(vec2 uv) {
        vec2 id = floor(uv);
        vec2 subuv = fract(uv);
        vec2 u = subuv * subuv * ( 2.0 - subuv );//subuv * subuv * (3. - 2. * subuv);//该方式应该是可以替换的，具体看你需要的效果
        float a = hash(id);
        float b = hash(id + vec2(1., 0.));
        float c = hash(id + vec2(0., 1.));
        float d = hash(id + vec2(1., 1.));
        return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
    }
    float fbm(in vec2 uv) {
        float s = .0;
        float m = .0;
        float a = .5;
        for(int i = 0; i < octaves; i++) {
        s += a * noise(uv);
        m += a;
        a *= .5;
        uv *= 2.;
        }
        return s / m;
    }

    vec3 domain(vec2 z){
        return vec3(hsb2rgb(vec3(atan(z.y,z.x)/TAU,1.,1.)));
    }
    vec3 colour(vec2 z) {
        return domain(z);
    }

    // The render function is where we render the pattern to be added to the layer
    vec3 render(vec2 uv, float scale, vec3 colour) {
        vec2 id = floor(uv);
        vec2 subuv = fract(uv);
        vec2 rand = hash2(id);
        float bokeh = abs(scale) * 1.;

        float particle = 0.;

        //用噪点图生成雪花粒子，若噪点上的一个点的vec2(r,b)的length>1.3，则这个地方被渲染为雪花
        if(length(rand) > 1.3) {
            vec2 pos = subuv-.5;
            float field = length(pos);
            particle = smoothstep(.3, 0., field); //这句话的语法在之前的版本是错误的？但是实际在chrome里又可以跑通。smoothstep的第二个值必须比第一个大
            particle += smoothstep(.4, 0.34 * bokeh, field); //
        }
        return vec3(particle*2.);
    }

    vec3 renderLayer(int layer, int layers, vec2 uv, inout float opacity, vec3 colour, float n) {
        vec2 _uv = uv;
        // Scale
        // Generating a scale value between zero and 1 based on a mod of u_time
        // A frequency of 10 dicided by the layer index (10 / layers * layer)
        // u_time和layer是不同的两个影响雪花效果的因素，每一层，会根据时间推移不断变大然后从最小开始，
        // 而layer相当于动画里的delay，用不同的层级，形成雪花多层次的感觉
        // 总体而言，这里的3d感官的雪花效果，并没有实际的3d雪花粒子坐标, scale范围永远是0 ~ 1
        float scale = mod((u_time + zoomSpeed / float(layers) * float(layer)) / zoomSpeed, -1.);
        uv *= 20.; // The initial scale. Increasing this makes the cells smaller and the "speed" apepar faster
        uv *= scale*scale; // then modifying the overall scale by the generated amount, scale越大，粒子越小
        // uv *= 1. + ( ( n*.5 ) * ( length(_uv) ) );
        // uv += .5*float(layer);
        //2d旋转粒子
        uv = rotate2d(u_time / 10.) * uv; // rotarting
        //为不同的层添加不同的初始位移，进一步差异化
        uv += vec2(25. + sin(u_time*.1)) * float(layer); // ofsetting the UV by an arbitrary amount to make the layer appear different

        // render
        vec3 pass = render(uv * multiplier, scale, colour) * .2; // render the pass

        // this is the opacity of the layer fading in from the "bottom"
        opacity = 1. + scale; //越大的粒子，透明度，越低
        float _opacity = opacity;

        // pass += n * .5 * mix(vec3(0., .5, 1.5), vec3(1., .5, 0.), opacity);

        // This is the opacity of the layer fading out at the top (we want this minimal, hence the smoothstep)
        float endOpacity = smoothstep(0., 0.4, scale * -1.);
        opacity += endOpacity;

        return pass * _opacity * endOpacity;
    }

    void main() {
        //将坐标从原始的x:0~width y:0~height 变成 x:-1~1 y:-1~1
        vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution.xy);

        if(u_resolution.y < u_resolution.x) {
            uv /= u_resolution.y;
        } else {
            uv /= u_resolution.x;
        }

        // uv.y += cos(u_time * .1) * .5;
        // uv.x += sin(u_time * .1) * .5;

        // float n = fbm(uv * 3. - 2.);
        //根据时间和当前坐标，在贴图上取对应的位置
        float n = fbm((uv + vec2(sin(u_time*.02), u_time*.02)) * 2. - 2.);

        vec3 colour = vec3(0.);
        colour = n * mix(vec3(.2, .4, .8), vec3(1.0, 1.0, 0.8), n);
        //这里用了clamp夹具函数后，相当于限制了第一个值vec3(1., .5, .25)*2. 的取值范围必须要在0到1之间, 等价于上面那个
        // colour = n * mix(vec3(0., .5, 1.5), clamp(vec3(1., .5, .25)*2., 0., 1.), n);
        colour -= n*n*n*n*.4;
        // colour += smoothstep(.8, 2.5, sin(n*n*n*8.))*.4;

        float opacity = 1.;
        float opacity_sum = 1.;

        for(int i = 1; i <= layers; i++) {
            colour += renderLayer(i, layers, uv, opacity, colour, n);
            opacity_sum += opacity;
        }

        colour /= opacity_sum;

        gl_FragColor = vec4(clamp(colour * 30., 0., 1.),1.0);
    }
`;

      let container;
      let camera, scene, renderer;
      let uniforms;

      let loader=new THREE.TextureLoader();
      let texture;
      loader.setCrossOrigin("anonymous");
      loader.load(
          'https://s3-us-west-2.amazonaws.com/s.cdpn.io/982762/noise.png',
          function do_something_with_texture(tex) {
            texture = tex;
            texture.wrapS = THREE.RepeatWrapping;
            texture.wrapT = THREE.RepeatWrapping;
            texture.minFilter = THREE.LinearFilter;
            init();
            animate();
          }
      );

      function init() {
        container = document.getElementById( 'container' );

        camera = new THREE.Camera();
        camera.position.z = 0;

        scene = new THREE.Scene();

        var geometry = new THREE.PlaneBufferGeometry( 2, 2 );

        uniforms = {
          u_time: { type: "f", value: 1.0 },
          u_resolution: { type: "v2", value: new THREE.Vector2() },
          u_noise: { type: "t", value: texture },
          u_mouse: { type: "v2", value: new THREE.Vector2() }
        };

        var material = new THREE.ShaderMaterial( {
          uniforms: uniforms,
          vertexShader: vertexShader,
          fragmentShader: fragmentShader
        } );
        material.extensions.derivatives = true;

        var mesh = new THREE.Mesh( geometry, material );
        scene.add( mesh );

        renderer = new THREE.WebGLRenderer();
        renderer.setPixelRatio( window.devicePixelRatio );

        container.appendChild( renderer.domElement );

        onWindowResize();
        window.addEventListener( 'resize', onWindowResize, false );

        document.addEventListener('pointermove', (e)=> {
          let ratio = window.innerHeight / window.innerWidth;
          uniforms.u_mouse.value.x = (e.pageX - window.innerWidth / 2) / window.innerWidth / ratio;
          uniforms.u_mouse.value.y = (e.pageY - window.innerHeight / 2) / window.innerHeight * -1;

          e.preventDefault();
        });
      }

      function onWindowResize( event ) {
        renderer.setSize( window.innerWidth, window.innerHeight );
        uniforms.u_resolution.value.x = renderer.domElement.width;
        uniforms.u_resolution.value.y = renderer.domElement.height;
      }

      function animate(delta) {
        requestAnimationFrame( animate );
        render(delta);
      }

      let then = 0;
      function render(delta) {

        uniforms.u_time.value = -10000 + delta * 0.0005;
        renderer.render( scene, camera );
      }
    }
  }
}
</script>

<style scoped>
#container{
  position:absolute;
  float: left;
  z-index: -999;
  width:100%;
  height:100%;
}
</style>