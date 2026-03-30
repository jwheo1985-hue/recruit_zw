# recruit_zw
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>채용 관리 에이전트 - APS 진천공장</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.4.120/pdf.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { font-family: 'Pretendard', sans-serif; }
        .drop-zone { border: 2px dashed #cbd5e1; transition: all 0.3s; }
        .drop-zone.active { border-color: #3b82f6; background-color: #eff6ff; }
    </style>
</head>
<body class="bg-slate-50 min-h-screen p-4 md:p-8">
    <div class="max-w-5xl mx-auto text-slate-800">
        <header class="mb-8 flex justify-between items-end">
            <div>
                <h1 class="text-3xl font-bold text-slate-900">APS 채용 관리 에이전트</h1>
                <p class="text-slate-500">이력서 분석 및 면접 일정 관리 시스템</p>
            </div>
            <div class="flex gap-2">
                <input type="password" id="apiKey" placeholder="Claude API Key 입력" class="border p-2 text-sm rounded w-48 shadow-sm">
            </div>
        </header>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <div class="md:col-span-1 bg-white p-6 rounded-xl shadow-sm border border-slate-200">
                <h2 class="text-lg font-semibold mb-4 border-b pb-2 text-blue-600">1. 이력서 분석</h2>
                <div id="dropZone" class="drop-zone p-8 rounded-lg text-center cursor-pointer mb-4">
                    <p class="text-sm text-slate-500">PDF 이력서를 여기에 끌어다 놓으세요</p>
                </div>
                <textarea id="rawText" class="hidden"></textarea>
                <button onclick="analyzeWithAI()" id="analyzeBtn" class="w-full bg-blue-600 text-white py-2 rounded-lg hover:bg-blue-700 disabled:bg-slate-300">Claude AI 요약하기</button>
            </div>

            <div class="md:col-span-2 bg-white p-6 rounded-xl shadow-sm border border-slate-200">
                <h2 class="text-lg font-semibold mb-4 border-b pb-2 text-blue-600">2. 후보자 현황</h2>
                <div id="candidateList" class="space-y-4">
                    </div>
            </div>
        </div>
    </div>

    <div id="msgModal" class="fixed inset-0 bg-black/50 hidden flex items-center justify-center p-4">
        <div class="bg-white p-6 rounded-xl max-w-lg w-full">
            <h3 class="text-xl font-bold mb-4">면접 안내 메시지</h3>
            <textarea id="msgContent" class="w-full h-64 border p-3 rounded-lg text-sm mb-4 bg-slate-50"></textarea>
            <div class="flex justify-end gap-2">
                <button onclick="closeModal()" class="px-4 py-2 border rounded-lg">닫기</button>
                <button onclick="copyMsg()" class="px-4 py-2 bg-green-600 text-white rounded-lg">복사하기</button>
            </div>
        </div>
    </div>

    <script>
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.4.120/pdf.worker.min.js';
        
        const dropZone = document.getElementById('dropZone');
        const candidateList = document.getElementById('candidateList');
        let candidates = JSON.parse(localStorage.getItem('aps_candidates') || '[]');

        function renderList() {
            candidateList.innerHTML = candidates.length ? '' : '<p class="text-center py-8 text-slate-400">등록된 후보자가 없습니다.</p>';
            candidates.forEach((c, index) => {
                const div = document.createElement('div');
                div.className = "border p-4 rounded-lg bg-slate-50 hover:shadow-md transition";
                div.innerHTML = `
                    <div class="flex justify-between mb-2">
                        <span class="font-bold text-lg text-slate-800">${c.name || '미확인'} 후보자</span>
                        <span class="text-sm text-slate-500">${new Date(c.date).toLocaleDateString()}</span>
                    </div>
                    <div class="text-sm text-slate-600 mb-4 whitespace-pre-line">${c.summary || '요약 정보 없음'}</div>
                    <div class="flex gap-2 border-t pt-3">
                        <button onclick="generateMessage(${index})" class="text-xs bg-slate-800 text-white px-3 py-1 rounded">면접 문자 생성</button>
                        <button onclick="deleteCandidate(${index})" class="text-xs border text-red-500 px-3 py-1 rounded ml-auto">삭제</button>
                    </div>
                `;
                candidateList.prepend(div);
            });
            localStorage.setItem('aps_candidates', JSON.stringify(candidates));
        }

        // PDF 텍스트 추출 로직
        dropZone.addEventListener('dragover', (e) => { e.preventDefault(); dropZone.classList.add('active'); });
        dropZone.addEventListener('dragleave', () => dropZone.classList.remove('active'));
        dropZone.addEventListener('drop', async (e) => {
            e.preventDefault();
            dropZone.classList.remove('active');
            const file = e.dataTransfer.files[0];
            if (file && file.type === 'application/pdf') {
                const reader = new FileReader();
                reader.onload = async function() {
                    const typedarray = new Uint8Array(this.result);
                    const pdf = await pdfjsLib.getDocument(typedarray).promise;
                    let fullText = '';
                    for (let i = 1; i <= pdf.numPages; i++) {
                        const page = await pdf.getPage(i);
                        const text = await page.getTextContent();
                        fullText += text.items.map(item => item.str).join(' ');
                    }
                    document.getElementById('rawText').value = fullText;
                    alert('PDF 텍스트 추출 완료! AI 요약을 진행하세요.');
                };
                reader.readAsArrayBuffer(file);
            }
        });

        // Claude AI 연동
        async function analyzeWithAI() {
            const apiKey = document.getElementById('apiKey').value;
            const text = document.getElementById('rawText').value;
            if(!apiKey) return alert('API Key를 입력해주세요.');
            if(!text) return alert('PDF를 먼저 업로드해주세요.');

            const btn = document.getElementById('analyzeBtn');
            btn.disabled = true;
            btn.innerText = "분석 중...";

            try {
                const response = await fetch('https://api.anthropic.com/v1/messages', {
                    method: 'POST',
                    headers: {
                        'x-api-key': apiKey,
                        'anthropic-version': '2023-06-01',
                        'content-type': 'application/json',
                        'dangerously-allow-browser': 'true'
                    },
                    body: JSON.stringify({
                        model: "claude-3-haiku-20240307",
                        max_tokens: 1000,
                        messages: [{
                            role: "user",
                            content: `다음 이력서 텍스트에서 1.이름 2.주요경력(요약) 3.보유기술 을 한국어로 추출해줘: ${text}`
                        }]
                    })
                });
                const data = await response.json();
                const aiResponse = data.content[0].text;
                const name = aiResponse.match(/이름:?\s*(\S+)/)?.[1] || "확인 불가";

                candidates.push({ name, summary: aiResponse, date: Date.now() });
                renderList();
                document.getElementById('rawText').value = '';
            } catch (error) {
                console.error(error);
                alert('AI 분석 중 오류가 발생했습니다.');
            } finally {
                btn.disabled = false;
                btn.innerText = "Claude AI 요약하기";
            }
        }

        // 면접 문자 생성
        function generateMessage(idx) {
            const c = candidates[idx];
            const msg = `안녕하세요, ${c.name}님. 
APS 진천공장(구 제니스월드) 채용 담당자입니다.

보내주신 이력서를 긍정적으로 검토하여 면접을 제안드리고자 연락드렸습니다.

■ 면접 일정: [날짜 및 시간 입력]
■ 면접 장소: 충북 진천군 [상세주소] (APS 진천공장)
■ 준비물: 신분증

참석 가능 여부를 회신 주시면 감사하겠습니다.`;
            document.getElementById('msgContent').value = msg;
            document.getElementById('msgModal').classList.remove('hidden');
        }

        function closeModal() { document.getElementById('msgModal').classList.add('hidden'); }
        function copyMsg() {
            const content = document.getElementById('msgContent');
            content.select();
            document.execCommand('copy');
            alert('복사되었습니다!');
        }
        function deleteCandidate(idx) {
            if(confirm('삭제하시겠습니까?')) {
                candidates.splice(idx, 1);
                renderList();
            }
        }

        renderList();
    </script>
</body>
</html>
