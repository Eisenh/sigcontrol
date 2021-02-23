<template>
 <div>
   
  <div  >
    <h6> Control of Power Supply with SCPI over WebUSBTMC</h6>
    <div class="q-gutter-md">
      <div class="row q-gutter-md ">
        
        <div class="col-xs-12  col-md-4 ">
          <q-card >
            <q-card-section class="q-gutter-md" >
              <div class="text-h6">Connect Device</div>
              <q-btn ref="connectButton"  @click="connectButtonClick()"> {{connectionState}} </q-btn> <output ref="devName" ></output>
              <br>
              <output id="notification-message">Notifications</output>
            </q-card-section>
          </q-card>
          
          <q-card >
            <q-card-section class="q-gutter-md">
              Run stops when either end condition is met. <br> Leave time at max to use accumulated coulombs as the end point.
              <br> 
              <input type="number" v-model="maxCoulombs" /> {{maxCoulombs}} C
              <input type="number" v-model="maxTimeS" /> {{maxTimeS}} S 
              
              <br> Read Interval (S) <input type="number" v-model="readIntervalS" /> {{readIntervalS}} S 
              <br>
            <q-btn ref="startBtn"  @click="startBtnClick()" >{{srtBtnLabel}} </q-btn>
            </q-card-section>
          </q-card>
           <q-card class="q-gutter-md">
            <div class="text-h6">Manual Commands</div>
            <q-card-section>
              <div class="row">
                <input type="text" ref="inputText" class="form-control" placeholder="Select or Enter Command" >  
                <q-btn id="wr-button" @click="writeBtnClick()">Write</q-btn> 
              </div>
            
            </q-card-section>
            <q-card-section>
                       <q-btn id="rd-button" @click="readManualResponse()">Read</q-btn>
                    
            </q-card-section>
          </q-card>
        </div>
        
        <div class="col-xs-12  col-md-7">
          <q-card>
            
            <q-card-section>
              
                        <div>
                         connected: {{isConnected}} <br>
                         running: {{isRunning}}  <br>
                         elapsed Time (ms): {{elapsedTimeM}} <br>
                         coulombs:  {{elapsedCoulombs}} <br>
                         current: {{current}} <br> 
                         <br>

                          <q-slider v-model="CH1ACalAdjust" :min="-0.003" :max="0.003"  :step="0.001"
      label
      :label-value="'Current Zero Adjust ' + CH1ACalAdjust + ' A'"
      label-always
      />  <br>
      
                           <q-slider v-model="cmdDelay" :min="100" :max="500" :step="10"  label
      :label-value="'Command Delay ' + cmdDelay + ' ms'"
      label-always/>
                        </div>
                         <br> <q-btn id="rd-button" @click="downloadData()">Download csvData</q-btn>
                        
                        <textarea  v-model="outputTextArea" rows=20 style="box-sizing: border-box; width: 90%; height= 90%" ></textarea>
                      
            </q-card-section>
          </q-card>
        </div>
        
      </div>
           
    </div>
  </div>
 
 </div>
</template>

<script>
// NOTE: use vanilla js version from webusbtmc folder as the source of code for this app
import Webusbtmc from '../assets/webusbtmc.js';


let tmc = new Webusbtmc();
export default {
  name: 'PageIndex',
  data(){
    return {
      srtBtnLabel: "Start",
      isConnected: false,
      isRunning: false,
      outputTextArea : "",
      startTime : 0,
      readIntervalS : 1,
      cmdDelay : 250,  // milliseconds between commands
      elapsedTimeM : 0,  // in milliS
      elapsedCoulombs: 0.0,
      maxTimeS : 15, // 86400,  // one day in milliS
      maxCoulombs : 100.0,
      CH1ACalAdjust : -0.002,
      current: 0,
      resultArray: [],
      csvHeader: " millis, accumC, twenty, V1, A1, W1, V2, A2, W2, setV1, setA1, setV2, setA2",
      csvData : this.csvHeader,
      lastRead: "",
      readTimer: null
    }
  },
  computed: {
      connectionState() { 
        return this.isConnected ? "Disconnect" : "Connect"
        },
      readInterval() {
        return this.readIntervalS * 1000
      },
      endCondition() {
        if (this.elapsedTimeM > (this.maxTimeS * 1000) || this.elapsedCoulombs > this.maxCoulombs) {
          console.log("endcondition ", this.elapsedTimeM , ", " , this.maxTimeS  , "," , this.elapsedCoulombs , "," ,  this.maxCoulombs)
          return true
        } else {
          return false
        }
      }
  },
  methods: {
 /*    setup() {
      tmc = new Webusbtmc();
      this.$refs.devName.innerHTML = "Connecting to " + JSON.stringify(tmc);
    }, */
    connectButtonClick() {
        console.log("in connectButtonClick() function");
        if (this.isConnected) {
            this.notify("");
            tmc.close();
            this.isConnected = false;
            this.$refs.devName.innerHTML = "    not connected";
            //this.$refs.connectButton.label = "Connect";
        } else {  // isConnected is true
            const filters = [
                { classCode: 0xfe, subclassCode: 0x03, protocolCode: 0x01 },
            ];
            navigator.usb
                .requestDevice({ filters: filters })
                .then((result) => {
                this.notify("");
                this.connect(result);
                })
                .catch((error) => {
                this.notify(error);
            });
        }
    },
    async initialize(){
      this.outputTextArea =  this.csvHeader ;
      this.elapsedTimeM = 0;
      this.elapsedCoulombs = 0;
      this.current = 0;
      this.resultArray= [];
      this.csvData = this.csvHeader;
      this.lastRead= "";
      this.startTime = Date.now();
      
      this.srtBtnLabel = "Stop";
      this.isRunning = true;
      this.writeDevice("OUTP CH1,ON");
      // setTimeout(this.readAll, this.cmdDelay);
       // blocking delay
        // first read 1 cmdDelay to write request, another to read result
    },
    startBtnClick(){
      if (this.isRunning) {
        
        this.endRun();
        setTimeout(() => {clearInterval(this.readTimer)}, 9000);  // fallback to max time limit
        
      } else {
        this.initialize();
        this.readTimer = setInterval(this.readAll, this.readIntervalS * 1000);  // subsequent reads
      }
    },
    async readAll( ){  // requests and reads each readInterval, then calls endRun()
      if (this.isConnected) {
        if (this.endCondition ) {
          this.endRun()
          } else {
              this.notify("");
              //let text = "*READALL?";
              this.writeDevice("*READALL?");
              setTimeout(this.readAllResponse, this.cmdDelay)  // reads after cmdDelay
              //console.log("Text is ", text, " read at  ", this.elapsedTimeM, "ms after a delay of ", this.cmdDelay)
          }
      } else {
        this.notify("trying to read while not Connected!");
        this.endRun();
      }
    },
    async readManualResponse(){
      
      let resultText =  await this.readDevice();  
      if (!resultText) {
        return;
      }
      this.outputTextArea = this.outputTextArea +  '\n' + resultText;

    },
    async readAllResponse(){
      let resultText =  await this.readDevice();
      this.resultArray = this.readAllParse(resultText);
      console.log("220 resultText is ", resultText);
      this.current = parseFloat(this.resultArray[3] ) + parseFloat(this.CH1ACalAdjust);   //TODO? - nake this an object?
      //let d = new Date()
      //let now = Date.now()
      this.elapsedTimeM = Date.now() - this.startTime;  //in milliS
      this.elapsedCoulombs =  this.elapsedCoulombs + this.readIntervalS * this.current;  //seconds
      
      console.log("218 resultText is ", resultText, " at ", this.elapsedTimeM, " with ", this.elapsedCoulombs, " C");
      this.addOutput(resultText);
      this.lastRead = resultText;
    },
    endRun(){
      this.notify("all done!");
      
      this.srtBtnLabel = "Start";
      this.isRunning = false;
      clearInterval(this.readTimer);
      setTimeout( this.writeDevice("OUTP CH1,OFF") , this.cmdDelay);  
      console.log("endRun()");
    },
    readAllParse(resultStr = "1,2") {  // expects a string of numbers
      
      console.log("233 result String is ", resultStr)  
      let resultArray = resultStr.split(',');
      console.log("result array is ", resultArray);
      let d = new Date()
      let now = d.getTime()
      resultArray.unshift(now);
     return  resultArray; 
    },
    writeDevice(text){
        console.log("writing at ", Date.now() - this.startTime, " ", text)
        if (this.isConnected) {
            if (text === undefined || text === '') {
              this.notify('Text is empty');
            }
            else {
                  tmc.write(text).then(() => {
                   console.log("cmd ", text);
                   // this.addOutput(text);
                  }).catch((error) => {
                    this.notify(error);
                  }) 
            }
        } else {
            this.notify('');
        }
      
    },
    writeBtnClick() {
       // console.log("in writeBtnClick() function ", this.isConnected);
        if (this.isConnected) {
            const text = this.$refs.inputText.value;

            if (text === undefined || text === '') {
              this.notify('Text is empty');
            }
            else {
              this.writeDevice(text);
              this.outputTextArea = this.outputTextArea +  '\n' + text;
            }
        } else {
            this.notify('not connected');
        }
      
    },

    async readDevice() {
        let resultText = "";
        console.log("reading at ", Date.now() - this.startTime, " ", this.isConnected);
        try {
          if (this.isConnected) {
            this.notify("");
            let length = 1024;

            await tmc
                .readBytes(length)
                .then((result) => {
                    //console.log(" after tmc read, result is ", result);
                    if (!result || result.length == 0) {
                      alert("Failed to receive the response");
                    } else {
                        
                      if (typeof result === 'string' || result instanceof String) {
                        resultText = result;
                        console.log("result was string ", result);
                      }
                      else {
                        //console.log("result wasn't string ", result);
                        const textDecoder = new TextDecoder();
                        resultText = textDecoder.decode(result);
                        //console.log("resultText before replacement >", resultText, "<");
                        resultText = resultText.replace(tmc.terminatorRead,'');
                        console.log("resultText after replacement >", resultText, "<");

                      } 
                    }
                })
                .catch((error) => {
                this.notify(error);
                });
            }
        }
      catch (err) {
        this.notify(error);
      }
      finally {
        return resultText;
      }
    },
    addOutput(text) {   // input must be a string, added to output text area and to csvData 
      if (!text) {
        return;
      }
      text = this.elapsedTimeM.toString() + ", " + this.elapsedCoulombs.toFixed(4) + ", " + text;
      console.log("in addOutput, text is ", text);
      this.csvData = this.csvData + '\n' + text;
      this.outputTextArea = this.outputTextArea +  '\n' + text;
    
    },
    
    objectToCsv(data){
      // get the headers from the first spot
      //const headers = Object.keys(data[0]);
      console.log("objtocsv data ", data);
      const headers = Object.keys(data[0]);  // an array
      const firstRow = 'Row,' + headers.join(',');
      const csvRows = [];
      csvRows.push(firstRow);

      // loop over rows
      for (const [i, row] of data.entries() ){
        const values = headers.map(header => {
          // escape any internal commas on fields converted to string (by ''+)
        const escaped =(''+ row[header]).replace(/"/g, '\\"');
        return  escaped;   //'"${escaped}"';
        });
        let rowdata = i + ',' + values.join(',');
        csvRows.push(rowdata);
      }
      return csvRows.join('\n');
    },
    
    downloadData(){
      
      const timestamp = Date.now();
      const fileName = timestamp + '.csv';       
      const downloadCsv = function(data, file) {
        const blob = new Blob([data], { type: 'text/csv' });
        const url = window.URL.createObjectURL(blob);
        const a = document.createElement('a');  // a blank element to generate the download url
        a.setAttribute('hidden', '');
        a.setAttribute('href', url);
        
        a.setAttribute('download', file);
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
      }; 
      downloadCsv(this.csvData, fileName);
      //this.downloadImage();
    },

    connect(device) {
      this.$refs.devName.innerHTML = "    Connecting to " + device.productName + "...";
      tmc
        .open(device)
        .then(() => {
          this.deviceName = device.productName.replace(/\s/g, "");
          this.$refs.devName.innerHTML = "    " + device.productName + " connected.";
          this.isConnected = true;
          //this.$refs.connectButton.label = "Disconnect";
        })
        .catch((error) => {
          this.$refs.devName.innerHTML = "-";
          this.notify(error);
        });
    },

    notify(message) {
        document.getElementById("notification-message").innerHTML = "Notifications: " + message;
    }

  },
  beforeUnmount() {
   // clearInterval(this.readInterval);
  },
}
</script>
