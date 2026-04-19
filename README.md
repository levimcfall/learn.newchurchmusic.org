<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Music XML Player</title>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js"></script>
    <script src="https://unpkg.com/opensheetmusicdisplay@1.8.8/build/opensheetmusicdisplay.min.js"></script>

    <style>
        body { font-family: system-ui, -apple-system, sans-serif; margin: 0; padding: 10px 20px 20px 20px; background-color: #f5f5f5; }
        
        .dashboard { 
            background: white; padding: 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0,0,0,0.15); 
            margin-bottom: 20px; position: sticky; top: 10px; z-index: 1000; max-height: 90vh; overflow-y: auto; 
        }
        
        .primary-controls { display: flex; flex-wrap: wrap; gap: 15px; align-items: center; border-bottom: 2px solid transparent; }
        .settings-panel { display: flex; flex-wrap: wrap; gap: 30px; margin-top: 15px; padding-top: 15px; border-top: 2px solid #eee; overflow: hidden; transition: 0.3s; }
        .settings-panel.hidden { max-height: 0; padding-top: 0; margin-top: 0; opacity: 0; }
        
        .control-group { display: flex; flex-direction: column; gap: 10px; }
        .row { display: flex; align-items: center; gap: 10px; }
        
        button { padding: 8px 16px; border: none; border-radius: 4px; background: #0066cc; color: white; cursor: pointer; font-weight: bold; transition: 0.2s; }
        button:hover:not(:disabled) { background: #0052a3; }
        button:disabled { background: #cccccc; cursor: not-allowed; }
        .small-btn { padding: 4px 8px; font-size: 12px; background: #555; }
        .small-btn:hover:not(:disabled) { background: #333; }
        
        select { padding: 6px; border-radius: 4px; border: 1px solid #ccc; width: 140px; }

        .mixer { border-left: 2px solid #eee; padding-left: 30px; flex-grow: 1; }
        .mixer h3 { margin-top: 0; font-size: 16px; color: #333; }
        .mixer-row { display: grid; grid-template-columns: 1.2fr 1fr 1.5fr; align-items: center; gap: 15px; margin-bottom: 12px; font-size: 14px; border-bottom: 1px solid #f0f0f0; padding-bottom: 8px; }

        #osmdCanvas { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); min-height: 400px; }
    </style>
</head>
<body>

    <div class="dashboard">
        <div class="primary-controls">
            <input type="file" id="fileInput" accept=".xml,.musicxml" />
            <button id="playBtn" disabled>Play</button>
            <button id="pauseBtn" disabled>Pause</button>
            <button id="stopBtn" disabled>Stop</button>
            <button id="seekBackBtn" disabled>&lt;&lt;&lt;</button>
            <button id="seekForwardBtn" disabled>&gt;&gt;&gt;</button>
            <button id="toggleSettingsBtn">Hide Settings</button>
        </div>

        <div class="settings-panel" id="settingsPanel">
            <div class="control-group">
                <div class="row">
                    <label style="font-weight: bold; width: 60px;">Tempo:</label>
                    <input type="range" id="tempoSlider" min="40" max="240" value="120" style="width: 100px;">
                    <span id="tempoValue" style="min-width: 30px;">120</span>
                    <button id="resetTempoBtn" class="small-btn" disabled>Reset</button>
                </div>
                <div class="row">
                    <label style="font-weight: bold; width: 110px;">Highlight Voice:</label>
                    <select id="voiceSelect" disabled><option value="none">None</option></select>
                </div>
                <div class="row">
                    <label style="cursor: pointer; margin-right: 15px;"><input type="checkbox" id="loopToggle"> Loop</label>
                    <label style="cursor: pointer; margin-right: 15px;"><input type="checkbox" id="metronomeToggle"> Metronome</label>
                    <label style="cursor: pointer;"><input type="checkbox" id="autoscrollToggle" checked> Autoscroll</label>
                </div>
            </div>

            <div class="mixer" id="mixerPanel">
                <h3>Voice Mixer</h3>
                <div style="color: #888; font-size: 13px;">Upload a file to begin...</div>
            </div>
        </div>
    </div>

    <div id="osmdCanvas"></div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const fileInput = document.getElementById('fileInput');
            const playBtn = document.getElementById('playBtn');
            const pauseBtn = document.getElementById('pauseBtn');
            const stopBtn = document.getElementById('stopBtn');
            const seekBackBtn = document.getElementById('seekBackBtn');
            const seekForwardBtn = document.getElementById('seekForwardBtn');
            const toggleSettingsBtn = document.getElementById('toggleSettingsBtn');
            const tempoSlider = document.getElementById('tempoSlider');
            const tempoValue = document.getElementById('tempoValue');
            const resetTempoBtn = document.getElementById('resetTempoBtn');
            const metronomeToggle = document.getElementById('metronomeToggle');
            const loopToggle = document.getElementById('loopToggle'); 
            const autoscrollToggle = document.getElementById('autoscrollToggle');
            const mixerPanel = document.getElementById('mixerPanel');
            const voiceSelect = document.getElementById('voiceSelect');
            const canvas = document.getElementById('osmdCanvas');

            const osmd = new opensheetmusicdisplay.OpenSheetMusicDisplay(canvas, { autoResize: true, backend: "svg" });

            let synths = {}; 
            let defaultBpm = 120; 
            let measureMap = []; 
            let isFirstEvent = true; 
            let highlightedElements = [];

            const patchLibrary = {
                "Standard Synth": { oscillator: { type: "triangle" }, envelope: { attack: 0.05, decay: 0.1, sustain: 0.3, release: 1 } },
                "Studio Piano": { oscillator: { type: "triangle8" }, envelope: { attack: 0.005, decay: 1.5, sustain: 0.1, release: 2 } },
                "Crystal Pluck": { oscillator: { type: "sawtooth" }, envelope: { attack: 0.001, decay: 0.2, sustain: 0, release: 0.2 } },
                "Warm Pad": { oscillator: { type: "sine" }, envelope: { attack: 0.8, decay: 0.5, sustain: 0.8, release: 2 } },
                "Square Lead": { oscillator: { type: "square" }, envelope: { attack: 0.01, decay: 0.1, sustain: 0.4, release: 0.5 } }
            };

            const clickSynth = new Tone.MembraneSynth({ pitchDecay: 0, envelope: { attack: 0, decay: 0.02, sustain: 0 } }).toDestination();
            clickSynth.volume.value = -10;

            // --- UI HANDLERS ---

            toggleSettingsBtn.addEventListener('click', () => {
                const isHidden = settingsPanel.classList.toggle('hidden');
                toggleSettingsBtn.textContent = isHidden ? 'Show Settings' : 'Hide Settings';
            });

            tempoSlider.addEventListener('input', (e) => {
                tempoValue.textContent = e.target.value;
                Tone.Transport.bpm.value = e.target.value;
            });

            loopToggle.addEventListener('change', (e) => {
                Tone.Transport.loop = e.target.checked;
            });

            resetTempoBtn.addEventListener('click', () => {
                tempoSlider.value = defaultBpm;
                tempoValue.textContent = defaultBpm;
                Tone.Transport.bpm.value = defaultBpm;
            });

            fileInput.addEventListener('change', (event) => {
                const file = event.target.files[0];
                if (!file) return;
                const reader = new FileReader();
                reader.onload = async (e) => {
                    stopPlayback();
                    await osmd.load(e.target.result);
                    osmd.render();

                    defaultBpm = 120; 
                    try {
                        const firstMeasure = osmd.sheet.SourceMeasures[0];
                        if (firstMeasure?.TempoExpressions) {
                            for (const expr of firstMeasure.TempoExpressions) {
                                if (expr.InstantaneousTempo?.TempoInBpm) {
                                    defaultBpm = Math.round(expr.InstantaneousTempo.TempoInBpm);
                                    break;
                                }
                            }
                        }
                    } catch(err) { console.warn("Tempo extraction failed, using 120."); }

                    tempoSlider.value = defaultBpm;
                    tempoValue.textContent = defaultBpm;
                    Tone.Transport.bpm.value = defaultBpm;
                    resetTempoBtn.disabled = false;

                    buildMixerUI(); 
                    playBtn.disabled = false;
                    voiceSelect.disabled = false;
                };
                reader.readAsText(file);
            });

            playBtn.addEventListener('click', async () => {
                await Tone.start();
                if (prepareAudioEvents()) {
                    Tone.Transport.start("+0.1");
                    playBtn.disabled = true; 
                    pauseBtn.disabled = false; 
                    stopBtn.disabled = false; 
                    seekBackBtn.disabled = false;
                    seekForwardBtn.disabled = false;
                }
            });

            pauseBtn.addEventListener('click', () => {
                if (Tone.Transport.state === 'started') { Tone.Transport.pause(); silenceSynths(); pauseBtn.textContent = 'Resume'; }
                else { Tone.Transport.start(); pauseBtn.textContent = 'Pause'; }
            });

            stopBtn.addEventListener('click', stopPlayback);

            seekBackBtn.addEventListener('click', () => {
                const currentIdx = osmd.cursor.Iterator.CurrentMeasureIndex;
                seekToMeasure(currentIdx - 1);
            });

            seekForwardBtn.addEventListener('click', () => {
                const currentIdx = osmd.cursor.Iterator.CurrentMeasureIndex;
                seekToMeasure(currentIdx + 1);
            });

            function seekToMeasure(targetMeasure) {
                if (!measureMap || measureMap.length === 0) return;
                
                // CRITICAL FIX: Silence all currently playing synths before we jump
                silenceSynths();
                
                // Clamp target index between 0 and the max measure index
                targetMeasure = Math.max(0, Math.min(measureMap.length - 1, targetMeasure));

                // Move Tone.js playback head
                Tone.Transport.position = (measureMap[targetMeasure] || 0) + "i";
                
                // Move visual cursor
                osmd.cursor.reset();
                while (!osmd.cursor.Iterator.EndReached && osmd.cursor.Iterator.CurrentMeasureIndex < targetMeasure) {
                    osmd.cursor.next();
                }
                
                isFirstEvent = true;
                clearHighlights();
                
                if (Tone.Transport.state === 'paused') {
                    pauseBtn.textContent = 'Resume';
                }
            }

            function stopPlayback() {
                Tone.Transport.stop(); Tone.Transport.position = 0;
                silenceSynths();
                if (osmd.cursor) { osmd.cursor.reset(); osmd.cursor.hide(); }
                clearHighlights();
                playBtn.disabled = false; 
                pauseBtn.disabled = true; 
                stopBtn.disabled = true; 
                seekBackBtn.disabled = true; 
                seekForwardBtn.disabled = true;
                pauseBtn.textContent = 'Pause';
            }

            function silenceSynths() { Object.values(synths).forEach(s => s.releaseAll()); }

            function clearHighlights() {
                highlightedElements.forEach(el => el.style.fill = el.dataset.originalFill || "");
                highlightedElements = [];
            }

            function getVoiceKey(note) {
                if (!note) return "fallback";
                let instrId = note.ParentVoiceEntry?.ParentVoice?.ParentInstrument?.Id ?? 
                             note.ParentVoice?.ParentInstrument?.Id ?? 
                             note.ParentStaff?.ParentInstrument?.Id;
                let voiceNum = note.ParentVoiceEntry?.ParentVoice?.VoiceId ?? 
                              note.ParentVoice?.VoiceId ?? 1;
                return (instrId !== undefined) ? `${instrId}-${voiceNum}` : "fallback";
            }

            function buildMixerUI() {
                mixerPanel.innerHTML = '<h3>Voice Mixer</h3>';
                voiceSelect.innerHTML = '<option value="none">None</option>';
                synths = {}; 

                osmd.sheet.Instruments.forEach(instr => {
                    const instrName = instr.Name || `Part ${instr.Id + 1}`;
                    let voiceIds = [];
                    if (instr.Voices) {
                        const vs = Array.isArray(instr.Voices) ? instr.Voices : Object.values(instr.Voices);
                        vs.forEach(v => { if(v && v.VoiceId !== undefined) voiceIds.push(v.VoiceId); });
                    }
                    if (voiceIds.length === 0) voiceIds.push(1);

                    [...new Set(voiceIds)].sort().forEach(voiceNum => {
                        const voiceKey = `${instr.Id}-${voiceNum}`;
                        const labelName = voiceIds.length > 1 ? `${instrName} (V${voiceNum})` : instrName;

                        synths[voiceKey] = new Tone.PolySynth(Tone.Synth, patchLibrary["Standard Synth"]).toDestination();
                        synths[voiceKey].volume.value = -8;

                        const row = document.createElement('div');
                        row.className = 'mixer-row';
                        const label = document.createElement('span'); label.textContent = labelName;

                        const sel = document.createElement('select');
                        Object.keys(patchLibrary).forEach(p => {
                            const o = document.createElement('option'); o.value = p; o.textContent = p; sel.appendChild(o);
                        });
                        sel.addEventListener('change', (e) => {
                            const vol = synths[voiceKey].volume.value;
                            synths[voiceKey].dispose();
                            synths[voiceKey] = new Tone.PolySynth(Tone.Synth, patchLibrary[e.target.value]).toDestination();
                            synths[voiceKey].volume.value = vol;
                        });

                        const sld = document.createElement('input');
                        sld.type = 'range'; sld.min = -40; sld.max = 5; sld.value = -8;
                        sld.addEventListener('input', (e) => {
                            synths[voiceKey].volume.value = e.target.value <= -40 ? -Infinity : e.target.value;
                        });

                        row.appendChild(label); row.appendChild(sel); row.appendChild(sld);
                        mixerPanel.appendChild(row);
                        
                        const opt = document.createElement('option'); opt.value = voiceKey; opt.textContent = labelName;
                        voiceSelect.appendChild(opt);
                    });
                });
                synths["fallback"] = new Tone.PolySynth(Tone.Synth, patchLibrary["Standard Synth"]).toDestination();
            }

            function prepareAudioEvents() {
                try {
                    Tone.Transport.cancel();
                    osmd.cursor.show(); osmd.cursor.reset();
                    Tone.Transport.bpm.value = tempoSlider.value;
                    const ppq = Tone.Transport.PPQ * 4;
                    measureMap = [];

                    let maxTick = 0; 

                    let isLoopWrap = false;
                    Tone.Transport.schedule((time) => {
                        Tone.Draw.schedule(() => {
                            if (isLoopWrap) {
                                osmd.cursor.reset();
                                isFirstEvent = true;
                                clearHighlights();
                                
                                if (autoscrollToggle.checked && osmd.cursor.cursorElement) {
                                    osmd.cursor.cursorElement.scrollIntoView({ behavior: 'auto', block: 'center' });
                                }
                            }
                            isLoopWrap = true; 
                        }, time);
                    }, 0);

                    while (!osmd.cursor.Iterator.EndReached) {
                        const ticks = Math.round(osmd.cursor.Iterator.currentTimeStamp.RealValue * ppq);
                        const mIdx = osmd.cursor.Iterator.CurrentMeasureIndex;
                        if (measureMap[mIdx] === undefined) measureMap[mIdx] = ticks;

                        const notes = osmd.cursor.NotesUnderCursor();
                        const currentTickEvents = {};

                        notes.forEach(n => {
                            const durationTicks = Math.round(n.Length.RealValue * ppq);
                            maxTick = Math.max(maxTick, ticks + durationTicks);

                            if (n.Pitch && !n.isRest()) {
                                const vKey = getVoiceKey(n);
                                if (!currentTickEvents[vKey]) currentTickEvents[vKey] = [];
                                currentTickEvents[vKey].push({ 
                                    p: Tone.Frequency(n.halfTone + 12, "midi").toNote(), 
                                    d: durationTicks + "i" 
                                });
                            }
                        });

                        Tone.Transport.schedule((time) => {
                            Tone.Draw.schedule(() => {
                                if (isFirstEvent) isFirstEvent = false; 
                                else if (!osmd.cursor.Iterator.EndReached) osmd.cursor.next();
                                
                                if (autoscrollToggle.checked && osmd.cursor.cursorElement) {
                                    osmd.cursor.cursorElement.scrollIntoView({ behavior: 'smooth', block: 'center' });
                                }

                                clearHighlights();
                                const selectedVoiceKey = voiceSelect.value;
                                if (selectedVoiceKey !== "none") {
                                    osmd.cursor.GNotesUnderCursor()?.forEach(gn => {
                                        if (getVoiceKey(gn.sourceNote) === selectedVoiceKey) {
                                            gn.getSVGGElement()?.querySelectorAll("path, rect")?.forEach(shape => {
                                                if (!shape.dataset.originalFill) shape.dataset.originalFill = shape.style.fill || shape.getAttribute("fill") || "";
                                                shape.style.fill = "#ff0000"; 
                                                highlightedElements.push(shape);
                                            });
                                        }
                                    });
                                }
                            }, time);

                            for (let vKey in currentTickEvents) {
                                currentTickEvents[vKey].forEach(note => {
                                    synths[vKey]?.triggerAttackRelease(note.p, note.d, time);
                                });
                            }
                        }, ticks + "i");

                        osmd.cursor.next();
                    }

                    osmd.cursor.reset(); isFirstEvent = true;
                    Tone.Transport.scheduleRepeat(t => { if (metronomeToggle.checked) clickSynth.triggerAttackRelease("G5", "64n", t); }, "4n");

                    Tone.Transport.loopStart = 0;
                    Tone.Transport.loopEnd = maxTick + "i";
                    Tone.Transport.loop = loopToggle.checked;

                    Tone.Transport.schedule((time) => {
                        Tone.Draw.schedule(() => {
                            if (!Tone.Transport.loop) {
                                stopPlayback();
                            }
                        }, time);
                    }, maxTick + "i");

                    return true;
                } catch (e) {
                    console.error("Audio Parsing Failed:", e);
                    return false;
                }
            }
        });
    </script>
</body>
</html>
