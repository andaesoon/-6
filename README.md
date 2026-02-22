<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>친목비 스마트 정산기</title>
    <!-- React 및 Tailwind CSS 로드 -->
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- 아이콘 로드 -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;700&display=swap');
        body { 
            background-color: #f8fafc; 
            font-family: 'Noto Sans KR', sans-serif; 
            -webkit-tap-highlight-color: transparent;
        }
        .loading-spinner {
            border: 4px solid rgba(59, 130, 246, 0.1);
            width: 40px; height: 40px; border-radius: 50%;
            border-left-color: #3b82f6; animation: spin 1s linear infinite;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
        .animate-in { animation: slideUp 0.4s ease-out; }
        @keyframes slideUp { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }
        
        /* 알림 메시지 스타일 */
        .toast-message {
            position: fixed;
            bottom: 2rem;
            left: 50%;
            transform: translateX(-50%);
            z-index: 50;
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect } = React;

        function App() {
            const [preview, setPreview] = useState(null);
            const [loading, setLoading] = useState(false);
            const [result, setResult] = useState(null);
            const [status, setStatus] = useState('');
            const [toast, setToast] = useState({ show: false, message: '', type: '' });

            // 제공해주신 Gemini API 키가 적용되었습니다.
            const apiKey = "AIzaSyAamHgTQJ6GUB7-HlUp7jQgpNPr8zib8Dk"; 

            // 토스트 메시지 표시 함수
            const showToast = (message, type = 'info') => {
                setToast({ show: true, message, type });
                setTimeout(() => setToast({ show: false, message: '', type: '' }), 3000);
            };

            // 1. 이미지 선택 핸들러
            const handleImageChange = (e) => {
                const file = e.target.files[0];
                if (file) {
                    const reader = new FileReader();
                    reader.onloadend = () => {
                        setPreview(reader.result);
                        setResult(null);
                        setStatus('');
                    };
                    reader.readAsDataURL(file);
                }
            };

            // 2. Gemini API를 이용한 영수증 분석
            const analyzeReceipt = async () => {
                if (!preview) return;
                if (!apiKey) {
                    showToast("API 키가 설정되지 않았습니다.", "error");
                    return;
                }
                
                setLoading(true);
                setStatus('AI가 영수증 정보를 추출하고 있습니다...');

                const base64Data = preview.split(',')[1];
                const prompt = `이 영수증 이미지에서 다음 정보를 추출해서 JSON 형식으로 응답해줘. 
                                1. date: 날짜 (YYYY-MM-DD 형식)
                                2. item: 상호명 또는 품목명 (최대한 짧게)
                                3. amount: 총 결제 금액 (숫자만)`;

                try {
                    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({
                            contents: [{
                                parts: [
                                    { text: prompt },
                                    { inlineData: { mimeType: "image/png", data: base64Data } }
                                ]
                            }],
                            generationConfig: {
                                responseMimeType: "application/json"
                            }
                        })
                    });

                    const data = await response.json();
                    
                    if (data.error) {
                        throw new Error(data.error.message);
                    }

                    const textResponse = data.candidates?.[0]?.content?.parts?.[0]?.text || "{}";
                    const parsed = JSON.parse(textResponse);
                    
                    setResult({
                        date: parsed.date || new Date().toISOString().split('T')[0],
                        item: parsed.item || '정산 항목',
                        amount: parsed.amount || 0,
                        type: '출금'
                    });
                    setStatus('분석 성공!');
                } catch (error) {
                    console.error(error);
                    setStatus('분석 중 오류가 발생했습니다.');
                    showToast("영수증 분석에 실패했습니다. API 키나 이미지를 확인해주세요.", "error");
                } finally {
                    setLoading(false);
                }
            };

            // 3. 구글 앱 스크립트 서버 함수 호출 (시트 저장)
            const saveToSheet = () => {
                if (typeof google === 'undefined' || !google.script) {
                    showToast("구글 앱 스크립트 환경이 아닙니다. 배포된 URL로 접속해주세요.", "error");
                    return;
                }

                setLoading(true);
                setStatus('구글 스프레드시트에 기록 중...');

                google.script.run
                    .withSuccessHandler((msg) => {
                        setLoading(false);
                        showToast(msg, "success");
                        setPreview(null);
                        setResult(null);
                    })
                    .withFailureHandler((err) => {
                        setLoading(false);
                        showToast("저장 실패: " + err, "error");
                    })
                    .addDataToSheet(result);
            };

            return (
                <div className="max-w-md mx-auto min-h-screen flex flex-col p-4 pb-20">
                    <header className="py-8 text-center">
                        <h1 className="text-3xl font-extrabold text-blue-600 tracking-tight flex items-center justify-center">
                            <i className="fas fa-receipt mr-3"></i>친목비 정산기
                        </h1>
                        <p className="text-gray-500 mt-2 text-sm">영수증 촬영으로 똑똑하게 회비 관리하기</p>
                    </header>

                    <main className="space-y-6 flex-grow">
                        {/* 업로드 영역 */}
                        <div className={`bg-white p-6 rounded-3xl shadow-sm border-2 border-dashed transition-all ${preview ? 'border-blue-400' : 'border-gray-300'} flex flex-col items-center justify-center relative`}>
                            {preview ? (
                                <div className="w-full">
                                    <img src={preview} className="w-full h-64 object-contain rounded-2xl mb-4" alt="영수증 미리보기" />
                                    <button onClick={() => {setPreview(null); setResult(null);}} className="absolute -top-3 -right-3 bg-red-500 text-white rounded-full w-10 h-10 flex items-center justify-center shadow-xl border-4 border-white">
                                        <i className="fas fa-times"></i>
                                    </button>
                                </div>
                            ) : (
                                <label className="cursor-pointer flex flex-col items-center py-10 w-full">
                                    <div className="w-20 h-20 bg-blue-50 text-blue-500 rounded-full flex items-center justify-center mb-4 transition-transform hover:scale-105 active:scale-95">
                                        <i className="fas fa-camera text-3xl"></i>
                                    </div>
                                    <span className="text-gray-600 font-bold text-lg">영수증 사진 올리기</span>
                                    <span className="text-gray-400 text-xs mt-2 italic text-center px-4 text-balance">영수증의 날짜와 금액을 AI가 자동으로 읽어옵니다</span>
                                    <input type="file" accept="image/*" className="hidden" onChange={handleImageChange} />
                                </label>
                            )}
                        </div>

                        {/* 분석 버튼 */}
                        {preview && !result && !loading && (
                            <button onClick={analyzeReceipt} className="w-full bg-blue-600 text-white py-5 rounded-2xl font-black text-lg shadow-xl hover:bg-blue-700 active:scale-95 transition-all">
                                <i className="fas fa-magic mr-2"></i>AI 영수증 분석 시작
                            </button>
                        )}

                        {/* 로딩 표시 */}
                        {loading && (
                            <div className="text-center py-10 bg-white rounded-3xl shadow-sm animate-pulse">
                                <div className="loading-spinner mx-auto mb-4"></div>
                                <p className="text-blue-600 font-bold">{status}</p>
                            </div>
                        )}

                        {/* 결과 데이터 편집창 */}
                        {result && !loading && (
                            <div className="bg-white p-8 rounded-3xl shadow-xl space-y-5 animate-in border-t-8 border-blue-500">
                                <h3 className="text-xl font-bold text-gray-800 flex items-center border-b pb-3">
                                    <i className="fas fa-edit mr-2 text-blue-500"></i>정산 내용 확인
                                </h3>
                                
                                <div className="space-y-4">
                                    <div className="grid grid-cols-1 gap-1">
                                        <label className="text-xs font-bold text-gray-400 uppercase tracking-wider ml-1">날짜</label>
                                        <input type="date" className="w-full bg-gray-50 border-0 rounded-xl p-4 focus:ring-2 focus:ring-blue-500" value={result.date} onChange={(e) => setResult({...result, date: e.target.value})} />
                                    </div>
                                    
                                    <div>
                                        <label className="text-xs font-bold text-gray-400 uppercase tracking-wider ml-1">항목명</label>
                                        <input type="text" className="w-full bg-gray-50 border-0 rounded-xl p-4 focus:ring-2 focus:ring-blue-500" value={result.item} onChange={(e) => setResult({...result, item: e.target.value})} />
                                    </div>
                                    
                                    <div>
                                        <label className="text-xs font-bold text-gray-400 uppercase tracking-wider ml-1">금액</label>
                                        <div className="relative">
                                            <input type="number" className="w-full bg-gray-50 border-0 rounded-xl p-4 font-bold text-blue-600 text-2xl focus:ring-2 focus:ring-blue-500 pr-12" value={result.amount} onChange={(e) => setResult({...result, amount: e.target.value})} />
                                            <span className="absolute right-4 top-1/2 -translate-y-1/2 font-bold text-gray-400">원</span>
                                        </div>
                                    </div>
                                    
                                    <div className="flex gap-3 pt-2">
                                        <button onClick={() => setResult({...result, type: '입금'})} className={`flex-1 py-4 rounded-xl font-bold transition-all ${result.type === '입금' ? 'bg-green-500 text-white shadow-lg scale-105' : 'bg-gray-100 text-gray-400'}`}>입금 (+)</button>
                                        <button onClick={() => setResult({...result, type: '출금'})} className={`flex-1 py-4 rounded-xl font-bold transition-all ${result.type === '출금' ? 'bg-red-500 text-white shadow-lg scale-105' : 'bg-gray-100 text-gray-400'}`}>출금 (-)</button>
                                    </div>
                                </div>
                                
                                <button onClick={saveToSheet} className="w-full bg-gray-900 text-white py-5 rounded-2xl font-black text-xl shadow-2xl mt-4 active:scale-95 transition-all">
                                    <i className="fas fa-check-circle mr-2"></i>구글 장부에 저장
                                </button>
                            </div>
                        )}
                    </main>
                    
                    {/* 토스트 알림 UI */}
                    {toast.show && (
                        <div className={`toast-message px-6 py-3 rounded-full shadow-2xl text-white font-bold animate-in flex items-center space-x-2 ${toast.type === 'success' ? 'bg-green-500' : toast.type === 'error' ? 'bg-red-500' : 'bg-gray-800'}`}>
                            <i className={`fas ${toast.type === 'success' ? 'fa-check-circle' : 'fa-info-circle'}`}></i>
                            <span>{toast.message}</span>
                        </div>
                    )}

                    <footer className="py-10 text-center">
                        <p className="text-gray-400 text-xs">스프레드시트 E열에 실시간 잔액이 계산됩니다.</p>
                    </footer>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
