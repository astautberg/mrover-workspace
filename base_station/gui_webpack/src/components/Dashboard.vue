<template>
  <div class="wrapper">
    <div class="box header">
      <img src="/static/mrover.png" alt="MRover" title="MRover" width="48" height="48" />
      <h1>Dashboard</h1>
      <div class="spacer"></div>
      <div class="comms">
        <ul id="vitals">
          <li><CommIndicator v-bind:connected="connections.websocket" name="Web Socket" /></li>
          <li><CommIndicator v-bind:connected="connections.lcm" name="Rover Connection Status" /></li>
          <li><CommIndicator v-bind:connected="connections.motors && connections.lcm" name="Driving" /></li>
        </ul>
      </div>
      <div class="spacer"></div>
      <div class="help">
        <img src="/static/help.png" alt="Help" title="Help" width="48" height="48" />
      </div>
      <div class="helpscreen"></div>
      <div class="helpimages" style="display: flex; align-items: center; justify-content: space-evenly">
        <img v-if="controlMode === 'arm'" src="/static/arm.png" alt="Robot Arm" title="Robot Arm Controls" style="width: auto; height: 70%; display: inline-block" />
        <img v-else-if="controlMode === 'soil_ac'" src="/static/soil_ac.png" alt="Soil Acquisition" title="Soil Acquisition Controls" style="width: auto; height: 70%; display: inline-block" />
        <img src="/static/joystick.png" alt="Joystick" title="Joystick Controls" style="width: auto; height: 70%; display: inline-block" />
      </div>
    </div>

    <div class="box odom light-bg">
      <OdometryReading v-bind:odom="odom"/>
    </div>
    <div class="box diags light-bg">
      <Sensors v-on:toggle-recording="onToggleRecording($event)" v-bind:rawData="sensors"/>
    </div>
    <div class="box cameras light-bg">
      <Cameras v-bind:servosData="lastServosMessage"/>
    </div>
    <div class="box map light-bg">
      <RoverMap v-bind:odom="odom"/>
    </div>
    <div class="box waypoints light-bg">
      <WaypointEditor v-bind:odom="odom" v-bind:nav_status="nav_status"/>
    </div>
    <div class="box controls light-bg">
      <Controls v-bind:dampen="dampen" v-bind:saMotor="saMotors"/>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapMutations } from 'vuex'
import Cameras from './Cameras.vue'
import RoverMap from './RoverMap.vue'
import CommIndicator from './CommIndicator.vue'
import OdometryReading from './OdometryReading.vue'
import Sensors from './Sensors.vue'
import Controls from './Controls.vue'
import WaypointEditor from './WaypointEditor.vue'
import L from '../leafletRover.js'
import LCMBridge from 'lcm_bridge_client/dist/bridge.js'

export default {
  name: 'Dashboard',
  data () {
    return {
      lcm_: null,

      controls: '',
      saMotors: {},
      dampen: 0,

      lastServosMessage: {
        pan: 0,
        tilt: 0
      },

      sensors: {
        temperature: 0,
        moisture: 0,
        conductivity: 0,
        pH: 0,
        O2: 0,
        CO2: 0,
        bcpu_temp: 0,
        gpu_temp: 0,
        tboard_temp: 0
      },

      odom: {
        latitude_deg: 38,
        latitude_min: 24.38226,
        longitude_deg: 110,
        longitude_min: 47.51724,
        bearing_deg: 0
      },

      connections: {
        websocket: false,
        lcm: false,
        motors: false
      },

      nav_status: {
        completed_wps: 0,
        missed_wps: 0,
        total_wps: 0
      }
    }
  },

  methods: {
    publish(channel, payload){
      this.lcm_.publish(channel, payload)
    }
  },

  computed: {
    ...mapGetters('autonomy', {
      autonEnabled: 'autonEnabled'
    }),

    ...mapGetters('controls', {
      controlMode: 'controlMode'
    }),
  },

  created: function () {
    this.lcm_ = new LCMBridge(
      'ws://localhost:8001',
      // Update WebSocket connection state
      (online) => {
        this.lcm_.setHomePage()
        this.connections.websocket = online
      },
      // Update connection states
      (online) => {
        this.connections.lcm = online[0]
      },
      // Subscribed LCM message received
      (msg) => {
        if (msg.topic === '/odom') {
          this.odom = msg.message
        } else if (msg.topic === '/sensors') {
          this.sensors = Object.assign(this.sensors, msg.message)
        } else if (msg.topic === '/temperature') {
          this.sensors = Object.assign(this.sensors, msg.message)
        } else if (msg.topic === '/kill_switch') {
          this.motors_active = !msg.message.killed
        } else if (msg.topic === '/nav_status') {
          this.nav_status = msg.message
        } else if (msg.topic === '/sa_motors') {
          this.saMotors = msg.message
        } else if (msg.topic === '/debugMessage') {
          if (msg['message']['isError']) {
            console.error(msg['message']['message'])
          } else {
            console.log(msg['message']['message'])
          }
        }
      },
      // Subscriptions
      [
        {'topic': '/odom', 'type': 'Odometry'},
        {'topic': '/sensors', 'type': 'Sensors'},
        {'topic': '/temperature', 'type': 'Temperature'},
        {'topic': '/kill_switch', 'type': 'KillSwitch'},
        {'topic': '/camera_servos', 'type': 'CameraServos'},
        {'topic': '/encoder', 'type': 'Encoder'},
        {'topic': '/nav_status', 'type': 'NavStatus'},
        {'topic': '/sa_motors', 'type': 'SAMotors'},
        {'topic': '/debugMessage', 'type': 'DebugMessage'}
      ]
    )

    const servosMessage = {
      'type': 'CameraServos',
      'pan': 0,
      'tilt': 0
    }

    const JOYSTICK_CONFIG = {
      'forward_back': 1,
      'left_right': 2,
      'dampen': 3,
      'kill': 4,
      'restart': 5,
      'pan': 4,
      'tilt': 5
    }

    const XBOX_CONFIG = {
      'left_js_x': 0,
      'left_js_y': 1,
      'left_trigger': 6,
      'right_trigger': 7,
      'right_js_x': 2,
      'right_js_y': 3,
      'right_bumper': 5,
      'left_bumper': 4,
      'd_pad_up': 12,
      'd_pad_down': 13
    }

    window.setInterval(() => {
      const gamepads = navigator.getGamepads()
      for (let i = 0; i < 2; i++) {
        const gamepad = gamepads[i]
        if (gamepad) {
          if (gamepad.id.includes('Logitech')) {
            const joystickData = {
              'type': 'Joystick',
              'forward_back': gamepad.axes[JOYSTICK_CONFIG['forward_back']],
              'left_right': gamepad.axes[JOYSTICK_CONFIG['left_right']],
              'dampen': gamepad.axes[JOYSTICK_CONFIG['dampen']],
              'kill': gamepad.buttons[JOYSTICK_CONFIG['kill']]['pressed'],
              'restart': gamepad.buttons[JOYSTICK_CONFIG['restart']]['pressed']
            }
            this.dampen = gamepad.axes[JOYSTICK_CONFIG['dampen']]

            if (!this.autonEnabled) {
              this.lcm_.publish('/drive_control', joystickData)
            }

            const servosSpeed = 0.8
            servosMessage['pan'] += gamepad.axes[JOYSTICK_CONFIG['pan']] * servosSpeed / 10
            servosMessage['tilt'] += -gamepad.axes[JOYSTICK_CONFIG['tilt']] * servosSpeed / 10
          } else if (gamepad.id.includes('Microsoft')) {
            const xboxData = {
              'type': 'Xbox',
              'left_js_x': gamepad.axes[XBOX_CONFIG['left_js_x']], // shoulder rotate
              'left_js_y': gamepad.axes[XBOX_CONFIG['left_js_y']], // shoulder tilt
              'left_trigger': gamepad.buttons[XBOX_CONFIG['left_trigger']]['value'], // elbow forward
              'right_trigger': gamepad.buttons[XBOX_CONFIG['right_trigger']]['value'], // elbow back
              'right_js_x': gamepad.axes[XBOX_CONFIG['right_js_x']], // hand rotate
              'right_js_y': gamepad.axes[XBOX_CONFIG['right_js_y']], // hand tilt
              'right_bumper': gamepad.buttons[XBOX_CONFIG['right_bumper']]['pressed'], // grip close
              'left_bumper': gamepad.buttons[XBOX_CONFIG['left_bumper']]['pressed'], // grip open
              'd_pad_up': gamepad.buttons[XBOX_CONFIG['d_pad_up']]['pressed'],
              'd_pad_down': gamepad.buttons[XBOX_CONFIG['d_pad_down']]['pressed']
            }
            if (this.controlMode === 'arm') {
              this.lcm_.publish('/arm_control', xboxData)
            } else if (this.controlMode === 'soil_ac') {
              this.lcm_.publish('/sa_control', xboxData)
            }
          }
        }
      }

      const clamp = function (num, min, max) {
        return num <= min ? min : num >= max ? max : num
      }

      servosMessage['pan'] = clamp(servosMessage['pan'], -1, 1)
      servosMessage['tilt'] = clamp(servosMessage['tilt'], -1, 1)
      this.lastServosMessage = servosMessage

      this.lcm_.publish('/camera_servos', servosMessage)
    }, 100)
  },

  components: {
    RoverMap,
    Cameras,
    CommIndicator,
    Sensors,
    Controls,
    OdometryReading,
    WaypointEditor
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .wrapper {
    display: grid;
    grid-gap: 10px;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 60px 2fr 1fr 1fr 2fr 70px 60px;
    grid-template-areas: "header header" "map diags" "map cameras" "map waypoints" "map waypoints" "controls waypoints" "odom waypoints";
    font-family: sans-serif;
    height: 98vh;
  }

  .box {
    border-radius: 5px;
    padding: 10px;
    border: 1px solid black;
  }

  .light-bg {
    background-color: LightGrey;
  }

  img {
    border: none;
    border-radius: 0px;
  }

  .header {
    grid-area: header;
    display: flex;
    align-items: center;
  }

  .header h1 {
    margin-left: 5px;
  }

  .spacer {
    flex-grow: 0.8;
  }

  .comms {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
  }

  .comms * {
    margin-top: 2px;
    margin-bottom: 2px;
  }

  .helpscreen {
    z-index: 1000000000;
    display: block;
    visibility: hidden;
    background-color: black;
    opacity: 0.8;
    position: absolute;
    left: 0px;
    top: 0px;
    width: 100%;
    height: 100%;
  }

  .helpimages {
    z-index: 1000000001;
    visibility: hidden;
    position: absolute;
    left: 5%;
    top: 5%;
    width: 90%;
    height: 90%;
  }

  .help {
    z-index: 1000000002;
    display: flex;
    float: right;
    opacity: 0.8;
    cursor: auto;
  }

  .help:hover {
    opacity: 1.0;
    cursor: pointer;
  }

  .help:hover ~ .helpscreen, .help:hover ~ .helpimages {
    visibility: visible;
  }

  .odom {
    grid-area: odom;
    font-size: 1em;
  }

  .diags {
    grid-area: diags;
  }

  .map {
    grid-area: map;
  }

  .waypoints {
    grid-area: waypoints;
  }

  .controls {
    grid-area: controls;
    font-size: 1em;
  }

  ul#vitals li {
    display: inline;
    float: left;
    padding: 0px 10px 0px 0px;
  }
</style>
