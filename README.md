< !DOCTYPE html >
    <html>

        <head>
            <meta charset="UTF-8">
                <title>Spring Boot Observability Dashboard</title>

                <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

                <style>

                    body {
                        font - family: Segoe UI;
                    background: #020617;
                    color: white;
                    padding: 20px;
}

                    h1 {
                        text - align: center;
                    margin-bottom: 20px;
}

                    .grid {
                        display: grid;
                    grid-template-columns: repeat(auto-fit, minmax(420px, 1fr));
                    gap: 20px;
}

                    .card {
                        background: #0f172a;
                    padding: 20px;
                    border-radius: 10px;
}

                    .title {
                        color: #38bdf8;
                    margin-bottom: 10px;
                    font-size: 18px;
}

                    pre {
                        background: black;
                    padding: 10px;
                    border-radius: 6px;
                    max-height: 300px;
                    overflow: auto;
}

                    canvas {
                        background: white;
                    border-radius: 5px;
}

                    button {
                        margin: 5px;
                    padding: 8px 12px;
                    border-radius: 5px;
                    cursor: pointer;
}

                </style>
        </head>

        <body>

            <h1>Spring Boot Observability Dashboard</h1>

            <div class="grid" id="grid"></div>

            <div class="card">
                <div class="title">Load Test Summary</div>
                <pre id="summary"></pre>
                <button onclick="exportJSON()">Export JSON</button>
                <button onclick="exportCSV()">Export Excel</button>
            </div>

            <div class="card">
                <div class="title">Automatic Bottleneck Detection</div>
                <pre id="bottleneck"></pre>
            </div>

            <div class="card">
                <div class="title">Top 5 Slow APIs</div>
                <pre id="slowApis"></pre>
            </div>

            <div class="card">
                <div class="title">HTTP Error Rate</div>
                <pre id="errorRate"></pre>
            </div>

            <div class="card">
                <div class="title">GC Pause Monitoring</div>
                <pre id="gcStats"></pre>
            </div>

            <div class="card">
                <div class="title">HTTP Spike Analyzer</div>
                <pre id="spikeAnalyzer"></pre>
            </div>

            <div class="card">
                <div class="title">Root Cause Timeline</div>
                <pre id="timeline"></pre>
            </div>

            <div class="card">
                <div class="title">Automatic Root Cause Predictor</div>
                <pre id="rootCause"></pre>
            </div>


            <script>

                const BASE="http://localhost:8081/actuator"

                let startTime=Date.now()

                let maxCPU=0
                let maxMemory=0
                let maxLatency=0
                let totalRequests=0
                let errorRate=0

                let gcPauseCount=0
                let gcPauseTime=0

                let slowApiList=[]
                let timelineEvents=[]


                async function getMetric(name){

try{
const r=await fetch(BASE+"/metrics/"+name)
                return await r.json()
}catch{
return null
}

}


                /* CPU */

                async function loadCPU(){

const usage=await getMetric("system.cpu.usage")

                if(!usage) return

                let cpu=(usage.measurements[0].value*100).toFixed(2)

if(cpu>maxCPU) maxCPU=cpu

}


                /* MEMORY */

                async function loadMemory(){

const used=await getMetric("jvm.memory.used")

                if(!used) return

                let usedMB=(used.measurements[0].value/1024/1024).toFixed(2)

if(usedMB>maxMemory) maxMemory=usedMB

}


                /* HTTP */

                async function loadHTTP(){

const m=await getMetric("http.server.requests")

                if(!m) return

const count=m.measurements.find(x=>x.statistic==="COUNT")?.value||0
const total=m.measurements.find(x=>x.statistic==="TOTAL_TIME")?.value||0

                let avg=0

if(count>0) avg=(total/count)*1000

if(avg>maxLatency) maxLatency=avg

                totalRequests=count

                detectLatencySpike(avg)

}


                /* ERROR RATE */

                async function loadErrorRate(){

const errors=await getMetric("http.server.requests?tag=status:500")
                const m=await getMetric("http.server.requests")

                if(!m) return

const total=m.measurements.find(x=>x.statistic==="COUNT")?.value||0
const err=errors?.measurements?.find(x=>x.statistic==="COUNT")?.value||0

if(total>0) errorRate=((err/total)*100).toFixed(2)

                document.getElementById("errorRate").innerHTML="HTTP Error Rate: "+errorRate+"%"

}


                /* GC */

                async function loadGC(){

const gc=await getMetric("jvm.gc.pause")

                if(!gc) return

gcPauseCount=gc.measurements.find(x=>x.statistic==="COUNT")?.value||0
gcPauseTime=gc.measurements.find(x=>x.statistic==="TOTAL_TIME")?.value||0

                document.getElementById("gcStats").innerHTML=
                "GC Pause Count: "+gcPauseCount+"\nGC Pause Time: "+gcPauseTime.toFixed(2)+" ms"

}


                /* SPIKE ANALYZER */

                function detectLatencySpike(value){

if(value>1000){

                    addTimeline("Latency spike detected")

                }

}


                /* TIMELINE */

                function addTimeline(event){

                    let time=new Date().toLocaleTimeString()

                timelineEvents.push(time+" - "+event)

                document.getElementById("timeline").innerHTML=timelineEvents.join("\n")

}


                /* ROOT CAUSE PREDICTOR */

                function predictRootCause(){

                    let cause="Unknown"
                let confidence=50

if(maxCPU>85){
                    cause = "CPU Saturation"
confidence=85
}

else if(maxLatency>1000 && errorRate<5){
                    cause = "Database or external dependency slowdown"
confidence=78
}

else if(errorRate>5){
                    cause = "Downstream service failure"
confidence=80
}

else if(gcPauseTime>500){
                    cause = "Memory pressure / GC pauses"
confidence=75
}

                document.getElementById("rootCause").innerHTML=
                "Predicted Root Cause: "+cause+"\nConfidence: "+confidence+"%"

}


                /* SUMMARY */

                function updateSummary(){

                    let duration=((Date.now()-startTime)/60000).toFixed(2)

                let report={
                    duration,
                    maxCPU,
                    maxMemory,
                    maxLatency,
                    totalRequests,
                    errorRate,
                    gcPauseCount,
                    gcPauseTime
                }

                window.reportData=report

                document.getElementById("summary").innerHTML=
                JSON.stringify(report,null,2)

}


                /* EXPORT JSON */

                function exportJSON(){

const data=JSON.stringify(window.reportData,null,2)

                const blob=new Blob([data],{type:"application/json"})

                const a=document.createElement("a")

                a.href=URL.createObjectURL(blob)
                a.download="load-test-report.json"
                a.click()

}


                /* EXPORT CSV */

                function exportCSV(){

const r=window.reportData

                let csv=
                `Duration, CPU, Memory, Latency, Requests, ErrorRate, GCCount, GCTime
                ${r.duration},${r.maxCPU},${r.maxMemory},${r.maxLatency},${r.totalRequests},${r.errorRate},${r.gcPauseCount},${r.gcPauseTime} `

                const blob=new Blob([csv],{type:"text/csv"})

                const a=document.createElement("a")

                a.href=URL.createObjectURL(blob)
                a.download="load-test-report.csv"
                a.click()

}


                /* MAIN LOOP */

                async function loadAll(){

                    await loadCPU()
await loadMemory()
                await loadHTTP()
                await loadErrorRate()
                await loadGC()

                predictRootCause()

                updateSummary()

}

                setInterval(loadAll,3000)

                loadAll()

            </script>

        </body>
    </html>

