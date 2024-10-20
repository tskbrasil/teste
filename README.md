```js
// ==UserScript==
// @name         cheat cmsp by iUnknown
// @namespace    https://cmspweb.ip.tv/
// @version      3.0
// @description  cheat cmsp
// @connect      cmsp.ip.tv
// @connect      edusp-api.ip.tv
// @author       iUnknown
// @match        https://cmsp.ip.tv/*
// @icon         https://edusp-static.ip.tv/permanent/66aa8b3a1454f7f2b47e21a3/full.x-icon
// @license      Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)
// ==/UserScript==

(function() {
    'use strict';

    let lesson_regex = /https:\/\/cmsp\.ip\.tv\/mobile\/tms\/task\/\d+\/apply/;
    console.log("-- cmsp cheat by iUnknown perdeu marcos --");

    function transformJson(jsonOriginal) {
        let novoJson = {
            status: "submitted",
            accessed_on: jsonOriginal.accessed_on,
            executed_on: jsonOriginal.executed_on,
            answers: {}
        };

        for (let questionId in jsonOriginal.answers) {
            let question = jsonOriginal.answers[questionId];
            let taskQuestion = jsonOriginal.task.questions.find(q => q.id === parseInt(questionId));

            if (taskQuestion.type === "order-sentences") {
                let answer = taskQuestion.options.sentences.map(sentence => sentence.value);
                novoJson.answers[questionId] = {
                    question_id: question.question_id,
                    question_type: taskQuestion.type,
                    answer: answer
                };
            } else if (taskQuestion.type === "text_ai") {
                let answer = taskQuestion.comment.replace(/<\/?p>/g, '');
                novoJson.answers[questionId] = {
                    question_id: question.question_id,
                    question_type: taskQuestion.type,
                    answer: { "0": answer }
                };
            } else if (taskQuestion.type === "fill-letters") {
                let answer = taskQuestion.options.answer;
                novoJson.answers[questionId] = {
                    question_id: question.question_id,
                    question_type: taskQuestion.type,
                    answer: answer
                };
            } else if (taskQuestion.type === "cloud") {
                let answer = taskQuestion.options.ids;
                novoJson.answers[questionId] = {
                    question_id: question.question_id,
                    question_type: taskQuestion.type,
                    answer: answer
                };
            } else {
                let answer = Object.fromEntries(
                    Object.keys(taskQuestion.options).map(optionId => [optionId, taskQuestion.options[optionId].answer])
                );
                novoJson.answers[questionId] = {
                    question_id: question.question_id,
                    question_type: taskQuestion.type,
                    answer: answer
                };
            }
        }
        return novoJson;
    }

    let oldHref = document.location.href;
    const observer = new MutationObserver(() => {
        if (oldHref !== document.location.href) {
            oldHref = document.location.href;
            if (lesson_regex.test(oldHref)) {
                console.log("[DEBUG] LESSON DETECTED");

                let x_auth_key = JSON.parse(sessionStorage.getItem("cmsp.ip.tv:iptvdashboard:state")).auth.auth_token;
                let room_name = JSON.parse(sessionStorage.getItem("cmsp.ip.tv:iptvdashboard:state")).room.room.name;
                let id = oldHref.split("/")[6];
                console.log(`[DEBUG] LESSON_ID: ${id} ROOM_NAME: ${room_name}`);

                let draft_body = {
                    status: "draft",
                    accessed_on: "room",
                    executed_on: room_name,
                    answers: {}
                };

                const sendRequest = (method, url, data, callback) => {
                    const xhr = new XMLHttpRequest();
                    xhr.open(method, url);
                    xhr.setRequestHeader("X-Api-Key", x_auth_key);
                    xhr.setRequestHeader("Content-Type", "application/json");
                    xhr.onload = () => callback(xhr);
                    xhr.onerror = () => console.error('Request failed');
                    xhr.send(data ? JSON.stringify(data) : null);
                };

                sendRequest("POST", `https://edusp-api.ip.tv/tms/task/${id}/answer`, draft_body, (response) => {
                    console.log("[DEBUG] DRAFT_DONE, RESPONSE: ", response.responseText);
                    let response_json = JSON.parse(response.responseText);
                    let task_id = response_json.id;
                    let get_anwsers_url = `https://edusp-api.ip.tv/tms/task/${id}/answer/${task_id}?with_task=true&with_genre=true&with_questions=true&with_assessed_skills=true`;

                    console.log("[DEBUG] Getting Answers...");

                    sendRequest("GET", get_anwsers_url, null, (response) => {
                        console.log(`[DEBUG] Get Answers request received response`);
                        console.log(`[DEBUG] GET ANSWERS RESPONSE: ${response.responseText}`);
                        let get_anwsers_response = JSON.parse(response.responseText);
                        let send_anwsers_body = transformJson(get_anwsers_response);

                        console.log(`[DEBUG] Sending Answers... BODY: ${JSON.stringify(send_anwsers_body)}`);

                        let delay = 10000; // Padrão de 10 segundos

                        setTimeout(() => {
                            sendRequest("PUT", `https://edusp-api.ip.tv/tms/task/${id}/answer/${task_id}`, send_anwsers_body, (response) => {
                                if (response.status !== 200) {
                                    alert(`[ERROR] An error occurred while sending the answers. RESPONSE: ${response.responseText}`);
                                }
                                console.log(`[DEBUG] Answers Sent! RESPONSE: ${response.responseText}`);

                                const watermark = document.querySelector('.MuiTypography-root.MuiTypography-body1.css-1exusee');
                                if (watermark) {
                                    watermark.textContent = 'Made by iUnknown';
                                    watermark.style.fontSize = '70px';
                                }
                            });
                        }, delay);

                        // Adiciona um botão para alterar a velocidade
                        const speedButton = document.createElement('button');
                        speedButton.textContent = 'Alterar Tempo de Finalização (ms)';
                        speedButton.style.position = 'fixed';
                        speedButton.style.top = '10px';
                        speedButton.style.right = '10px';
                        speedButton.style.zIndex = '9999';
                        speedButton.style.padding = '10px';
                        speedButton.style.backgroundColor = '#4CAF50';
                        speedButton.style.color = 'white';
                        speedButton.style.border = 'none';
                        speedButton.style.borderRadius = '5px';
                        document.body.appendChild(speedButton);

                        speedButton.onclick = function() {
                            const newDelay = prompt("Insira o novo tempo de finalização (em milissegundos):", "10000");
                            if (newDelay !== null && !isNaN(newDelay) && newDelay > 0) {
                                delay = parseInt(newDelay);
                                alert(`Tempo de finalização alterado para ${delay} ms.`);
                            } else {
                                alert("Por favor, insira um tempo válido.");
                            }
                        };
                    });
                });
            }
        }
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });

    // Adicionando a telinha preta com texto
    function createOverlay() {
        const overlay = document.createElement('div');
        overlay.style.position = 'fixed';
        overlay.style.top = '0';
        overlay.style.left = '0';
        overlay.style.width = '100%';
        overlay.style.height = '100%';
        overlay.style.backgroundColor = 'rgba(0, 0, 0, 0.8)';
        overlay.style.zIndex = '9999';
        overlay.style.display = 'flex';
        overlay.style.justifyContent = 'center';
        overlay.style.alignItems = 'center';
        overlay.style.color = 'white';
        overlay.style.fontSize = '20px';
        overlay.style.fontFamily = 'Arial, sans-serif';
        overlay.innerHTML = `
            <div style="text-align: center;">
                <p>Entre no nosso Discord!</p>
                <a href="https://discord.gg/2rqNgcZG66" target="_blank" style="display: inline-block; margin-top: 10px; padding: 10px 20px; background-color: #7289DA; color: white; text-decoration: none; border-radius: 5px;">Acessar Discord</a>
            </div>
        `;

        document.body.appendChild(overlay);

        // Fecha a telinha automaticamente após 10 segundos
        setTimeout(() => {
            document.body.removeChild(overlay);
        }, 10000);
    }

    // Chama a função para criar a telinha
    createOverlay();

})();

``` 
