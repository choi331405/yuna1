<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>유나</title>
  <style>
    body {
      background: #000;
      font-family: 'Arial', sans-serif;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
      padding: 20px;
      gap: 20px;
    }
    .timer {
      background: #111;
      padding: 20px;
      border-radius: 15px;
      text-align: center;
      width: 250px;
      box-shadow: 0 0 20px rgba(0,255,255,0.6);
    }
    .title {
      font-size: 1.2em;
      margin-bottom: 10px;
      color: #0ff;
      text-shadow: 0 0 10px #0ff, 0 0 20px #0ff;
      cursor: pointer;
    }
    .title-input {
      width: 140px;
      background: #000;
      border: none;
      outline: none;
      text-align: center;
      font-size: 1.2em;
      color: #0ff;
      text-shadow: 0 0 10px #0ff, 0 0 20px #0ff;
    }
    .time {
      font-size: 2em;
      color: #0ff;
      text-shadow: 0 0 10px #0ff, 0 0 20px #0ff;
    }
    .time-part {
      cursor: pointer;
      padding: 0 2px;
    }
    .time-input {
      width: 30px;
      background: #000;
      border: none;
      outline: none;
      text-align: center;
      font-size: 1em;
      color: #0ff;
      text-shadow: 0 0 10px #0ff, 0 0 20px #0ff;
    }
    .end-time {
      font-size: 0.9em;
      margin-top: 8px;
      color: #ff0;
      text-shadow: 0 0 10px #ff0, 0 0 20px #ff0;
    }
    button {
      margin: 5px;
      padding: 8px 15px;
      border: none;
      border-radius: 8px;
      font-size: 1em;
      cursor: pointer;
      background: #222;
      color: #0ff;
      text-shadow: 0 0 5px #0ff;
      transition: 0.2s;
    }
    button:hover {
      background: #0ff;
      color: #000;
    }
    #copyArea {
      width: 100%;
      text-align: center;
      margin-top: 20px;
    }
    #preview {
      margin-top: 10px;
      color: #0ff;
      font-size: 1em;
      text-shadow: 0 0 5px #0ff, 0 0 10px #0ff;
    }
  </style>
</head>
<body>
  <audio id="beep" src="https://actions.google.com/sounds/v1/alarms/beep_short.ogg"></audio>
  <div id="copyArea">
    <button onclick="copyActiveTimers()">활성 타이머 복사</button>
    <div id="preview">활성화된 타이머 없음</div>
  </div>
  <script>
    let activeTimers = [];

    class NeonTimer {
      constructor(container, index, savedData) {
        this.index = index + 1;
        this.name = savedData?.name || `타이머 ${this.index}`;
        this.totalSeconds = savedData?.totalSeconds || 0;
        this.remainingSeconds = this.totalSeconds;

        this.title = document.createElement("span");
        this.title.className = "title";
        this.title.textContent = this.name;
        this.title.addEventListener("click",()=> this.editTitle());

        this.timeDisplay = document.createElement("div");
        this.timeDisplay.className = "time";
        this.updateTimeDisplay();

        this.endTimeDisplay = document.createElement("div");
        this.endTimeDisplay.className = "end-time";

        this.startBtn = this.createButton("시작", () => this.start());
        this.pauseBtn = this.createButton("일시정지", () => this.pause());
        this.resetBtn = this.createButton("리셋", () => this.reset());

        this.container = document.createElement("div");
        this.container.className = "timer";
        this.container.appendChild(this.title);
        this.container.appendChild(this.timeDisplay);
        this.container.appendChild(this.endTimeDisplay);
        this.container.appendChild(this.startBtn);
        this.container.appendChild(this.pauseBtn);
        this.container.appendChild(this.resetBtn);

        container.appendChild(this.container);
        this.interval = null;
      }

      createButton(label, action) {
        let btn = document.createElement("button");
        btn.textContent = label;
        btn.addEventListener("click", action);
        return btn;
      }

      formatTime(sec) {
        let h = String(Math.floor(sec / 3600)).padStart(2, "0");
        let m = String(Math.floor((sec % 3600) / 60)).padStart(2, "0");
        let s = String(sec % 60).padStart(2, "0");
        return [h, m, s];
      }

      updateTimeDisplay() {
        let [h, m, s] = this.formatTime(this.totalSeconds);
        this.timeDisplay.innerHTML = "";
        this.addTimePart("h", h);
        this.timeDisplay.appendChild(document.createTextNode(":"));
        this.addTimePart("m", m);
        this.timeDisplay.appendChild(document.createTextNode(":"));
        this.addTimePart("s", s);
      }

      addTimePart(type, val) {
        let span = document.createElement("span");
        span.className = "time-part";
        span.textContent = val;
        span.addEventListener("click", () => this.editTimePart(type, span));
        this.timeDisplay.appendChild(span);
      }

      editTitle() {
        let input = document.createElement("input");
        input.type = "text";
        input.value = this.title.textContent;
        input.className = "title-input";
        this.title.replaceWith(input);
        input.focus();

        const save = () => {
          this.name = input.value || "타이머";
          this.title.textContent = this.name;
          input.replaceWith(this.title);
          this.title.addEventListener("click", () => this.editTitle());
          saveAllTimers();
        };
        input.addEventListener("blur", save);
        input.addEventListener("keydown", e => {
          if (e.key === "Enter") save();
        });
      }

      editTimePart(type, span) {
        let input = document.createElement("input");
        input.type = "text";
        input.value = span.textContent;
        input.maxLength = 2;
        input.className = "time-input";
        span.replaceWith(input);
        input.focus();

        const save = () => {
          let val = input.value.padStart(2,"0");
          let [h, m, s] = this.formatTime(this.totalSeconds).map(Number);

          if (type === "h") h = Math.min(23, Math.max(0, Number(val)));
          if (type === "m") m = Math.min(59, Math.max(0, Number(val)));
          if (type === "s") s = Math.min(59, Math.max(0, Number(val)));

          this.totalSeconds = h*3600 + m*60 + s;
          this.remainingSeconds = this.totalSeconds;
          this.updateTimeDisplay();
          this.endTimeDisplay.textContent = "";
          saveAllTimers();
        };
        input.addEventListener("blur", save);
        input.addEventListener("keydown", e => {
          if (e.key === "Enter") save();
        });
      }

      start() {
        if (this.remainingSeconds <= 0) return;
        if (this.interval) return;

        let endTime = new Date(Date.now() + this.remainingSeconds * 1000);
        let hh = String(endTime.getHours()).padStart(2,"0");
        let mm = String(endTime.getMinutes()).padStart(2,"0");
        this.endTimeDisplay.textContent = "종료 예정 시각: " + hh + ":" + mm;

        if (!activeTimers.includes(this)) activeTimers.push(this);
        updatePreview();

        this.interval = setInterval(() => {
          this.remainingSeconds--;
          let [h, m, s] = this.formatTime(this.remainingSeconds);
          this.timeDisplay.innerHTML = "";
          this.addTimePart("h", h);
          this.timeDisplay.appendChild(document.createTextNode(":"));
          this.addTimePart("m", m);
          this.timeDisplay.appendChild(document.createTextNode(":"));
          this.addTimePart("s", s);

          if (this.remainingSeconds <= 0) {
            clearInterval(this.interval);
            this.interval = null;
            document.getElementById("beep").play();
            this.reset();
          }
        }, 1000);
      }

      pause() {
        clearInterval(this.interval);
        this.interval = null;
        activeTimers = activeTimers.filter(t => t !== this);
        updatePreview();
      }

      reset() {
        clearInterval(this.interval);
        this.interval = null;
        this.remainingSeconds = this.totalSeconds;
        this.updateTimeDisplay();
        this.endTimeDisplay.textContent = "";
        activeTimers = activeTimers.filter(t => t !== this);
        updatePreview();
      }

      getData() {
        return {
          name: this.name,
          totalSeconds: this.totalSeconds
        };
      }
    }

    function saveAllTimers() {
      let data = timers.map(t => t.getData());
      localStorage.setItem("neonTimers", JSON.stringify(data));
    }

    function loadAllTimers() {
      return JSON.parse(localStorage.getItem("neonTimers") || "[]");
    }

    /* ----------------------------------------------------
       ✔ 완전 수정된 부분 (공백 제거 + trim 추가)
    ---------------------------------------------------- */
    function updatePreview() {
      let normal = activeTimers.filter(t => t.index !== 6);
      let last = activeTimers.find(t => t.index === 6);

      let text = normal
        .map(t => {
          let time = t.endTimeDisplay.textContent
            .replace("종료 예정 시각: ","")
            .trim();   // ← 공백 완전 제거
          return `${t.name}(${time})`;
        })
        .join("/");   // ← 공백 없는 /

      if (last) {
        let lastTime = last.endTimeDisplay.textContent
          .replace("종료 예정 시각: ","")
          .trim();    // ← 공백 제거

        text = text ?
          `${text}//${last.name}(${lastTime})` :
          `${last.name}(${lastTime})`;
      }

      document.getElementById("preview").textContent =
        text || "활성화된 타이머 없음";
    }

    /* ----------------------------------------------------
       ✔ 복사 기능도 동일하게 수정됨
    ---------------------------------------------------- */
    function copyActiveTimers() {
      let normal = activeTimers.filter(t => t.index !== 6);
      let last = activeTimers.find(t => t.index === 6);

      let text = normal
        .map(t => {
          let time = t.endTimeDisplay.textContent
            .replace("종료 예정 시각: ","")
            .trim();   // ← 공백 제거
          return `${t.name}(${time})`;
        })
        .join("/");  // ← 슬래시 앞뒤 공백 없음

      if (last) {
        let lastTime = last.endTimeDisplay.textContent
          .replace("종료 예정 시각: ","")
          .trim();
        text = text ?
          `${text}//${last.name}(${lastTime})` :
          `${last.name}(${lastTime})`;
      }

      if (!text) text = "활성화된 타이머 없음";
      navigator.clipboard.writeText(text);
    }

    let saved = loadAllTimers();
    let timers = [];
    for (let i = 0; i < 6; i++) {
      timers.push(new NeonTimer(document.body, i, saved[i]));
    }
  </script>
</body>
</html>
