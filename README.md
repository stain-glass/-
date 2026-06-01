import React, { useState, useEffect } from 'react';
import { 
  Compass, 
  Map, 
  Sparkles, 
  RotateCcw, 
  ArrowRight, 
  BookOpen, 
  CheckCircle2, 
  Volume2, 
  VolumeX, 
  User, 
  Layers, 
  Award,
  BookMarked,
  Scroll,
  ChevronRight,
  CloudSun
} from 'lucide-react';

// 국악 감성의 오음계 음향 효과 신디사이저
const playTraditionalTone = (toneType) => {
  try {
    const AudioContext = window.AudioContext || window.webkitAudioContext;
    if (!AudioContext) return;
    const ctx = new AudioContext();
    
    const pentatonic = {
      gung: 261.63, // C4
      sang: 293.66, // D4
      gak: 329.63,  // E4
      chi: 392.00,  // G4
      woo: 440.00,  // A4
      highGung: 523.25 // C5
    };

    const playTone = (freq, type, duration, delay = 0) => {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.type = type;
      osc.frequency.setValueAtTime(freq, ctx.currentTime + delay);
      
      gain.gain.setValueAtTime(0.08, ctx.currentTime + delay);
      gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + delay + duration);
      
      osc.connect(gain);
      gain.connect(ctx.destination);
      osc.start(ctx.currentTime + delay);
      osc.stop(ctx.currentTime + delay + duration);
    };

    if (toneType === 'click') {
      playTone(pentatonic.chi, 'sine', 0.25);
    } else if (toneType === 'success') {
      playTone(pentatonic.gung, 'triangle', 0.3, 0);
      playTone(pentatonic.gak, 'triangle', 0.3, 0.1);
      playTone(pentatonic.chi, 'triangle', 0.3, 0.2);
      playTone(pentatonic.highGung, 'triangle', 0.5, 0.3);
    } else if (toneType === 'mystic') {
      playTone(pentatonic.woo, 'sine', 0.8, 0);
      playTone(pentatonic.chi, 'sine', 1.0, 0.25);
    } else if (toneType === 'ending') {
      const notes = [pentatonic.gung, pentatonic.sang, pentatonic.gak, pentatonic.chi, pentatonic.woo, pentatonic.highGung];
      notes.forEach((freq, idx) => {
        playTone(freq, 'sine', 0.6, idx * 0.15);
      });
    }
  } catch (e) {
    console.log("Audio Context not supported.");
  }
};

export default function App() {
  // 게임 진행 단계 (0: 조선 지도 전도, 1: 공간 거점 활성화, 2: 남원 기린산 안개, 3: 만복사 삼교 분기, 4: 부벽정 퍼즐, 5: 사유록 책자, 6: 유람 완수)
  const [currentStage, setCurrentStage] = useState(0);
  const [selectedEnding, setSelectedEnding] = useState(null); // 'confucian' | 'buddhist' | 'taoist'
  const [isMuted, setIsMuted] = useState(false);
  
  // 미니게임 1: 남원 기린산 안개 걷어내기
  const [fogPixels, setFogPixels] = useState(Array(12).fill(true));
  const [fogCleared, setFogCleared] = useState(false);
  
  // 미니게임 2: 평양 부벽정 기억 맞추기
  const [puzzleConnected, setPuzzleConnected] = useState({
    eulmildae: null, 
    daedongriver: null,
    bubyeokjeong: null
  });
  const [selectedPiece, setSelectedPiece] = useState(null);
  const [puzzleSolved, setPuzzleSolved] = useState(false);

  // 화면에 보이지 않고 오직 한국어 음성으로만 출력될 나레이션 대사 세트
  const narrationVoices = {
    stage0: "일천 사백 오십 오년, 한 지식인의 위대한 방랑이 시작되었습니다. 세상살이의 소란스러움을 뒤로하고 영혼의 자취를 찾아 떠난 선비의 여정이 고요히 열립니다.",
    stage1: "당신의 고결한 발걸음이 향할 조선의 세 거점이 찬란한 은빛 안개 속에서 고요히 깨어납니다. 지도를 누르면 새로운 기록의 장이 펼쳐집니다.",
    stage2: "외로운 선비 양생의 애달픈 발자취를 따라 기린산에 서린 구름 안개를 걷어내십시오. 손끝이 닿을 때마다 숨겨진 고찰 만복사가 모습을 드러냅니다.",
    stage3_intro: "이별의 순간이 새벽녘 안개 사이로 다가옵니다. 이룰 수 없는 사랑의 끝에서 당신은 과연 어떤 지혜로운 가치를 선택하시겠습니까?",
    stage3_confucian: "양생은 떠난 귀녀의 은그릇을 온 가슴으로 껴안은 채 평생 절개를 지키며 의로운 사대부의 높은 정절을 지켜내기로 결심합니다.",
    stage3_buddhist: "부질없는 집착을 비워내고, 저 너머 고요한 환생의 수레바퀴 속에서 고통 없는 윤회의 지혜를 은은하게 받아들입니다.",
    stage3_taoist: "현실의 낡고 허무한 도리들을 아낌없이 던져둔 채, 기린산의 안개를 벗 삼아 깊은 지리산 자락으로 들어가 영원한 신선이 됩니다.",
    stage4: "맑고 푸른 대동강 물결 위에 잠든 인물들의 기억을 본래의 역사가 서린 올바른 장소에 놓아, 방랑하는 영혼들의 원한을 보듬어 주십시오.",
    stage5: "당신이 조선 팔도를 헤매며 보듬어 낸 슬픔과 지혜가 하나로 엮여, 김시습의 불후의 소설이자 방랑의 고귀한 성찰인 사유록 책자로 완정됩니다.",
    stage6: "동양의 아늑한 우주관과 깊은 사랑의 지혜를 함께 유람해 주셔서 진심으로 감사드립니다. 당신의 모든 사유는 이 책에 고스란히 영원히 남을 것입니다."
  };

  // TTS 한국어 음성 재생 함수 (화면 자막은 일체 없음)
  const speakVoice = (text) => {
    if (isMuted) return;
    try {
      if ('speechSynthesis' in window) {
        window.speechSynthesis.cancel(); // 진행 중인 이전 음성 중단
        const utterance = new SpeechSynthesisUtterance(text);
        utterance.lang = 'ko-KR';
        utterance.rate = 0.95;  // 동양적인 서정성을 살리기 위해 미세하게 느린 속도
        utterance.pitch = 1.0;
        window.speechSynthesis.speak(utterance);
      }
    } catch (e) {
      console.warn("Speech synthesis error: ", e);
    }
  };

  // 스테이지가 바뀔 때마다 자동으로 해당 나레이션 음성 구동
  useEffect(() => {
    if (currentStage === 0) {
      speakVoice(narrationVoices.stage0);
    } else if (currentStage === 1) {
      speakVoice(narrationVoices.stage1);
    } else if (currentStage === 2) {
      speakVoice(narrationVoices.stage2);
    } else if (currentStage === 3) {
      if (!selectedEnding) {
        speakVoice(narrationVoices.stage3_intro);
      }
    } else if (currentStage === 4) {
      speakVoice(narrationVoices.stage4);
    } else if (currentStage === 5) {
      speakVoice(narrationVoices.stage5);
    } else if (currentStage === 6) {
      speakVoice(narrationVoices.stage6);
    }
  }, [currentStage]);

  // 엔딩 선택 시의 음성 변경 감지
  useEffect(() => {
    if (currentStage === 3 && selectedEnding) {
      if (selectedEnding === 'confucian') speakVoice(narrationVoices.stage3_confucian);
      if (selectedEnding === 'buddhist') speakVoice(narrationVoices.stage3_buddhist);
      if (selectedEnding === 'taoist') speakVoice(narrationVoices.stage3_taoist);
    }
  }, [selectedEnding, currentStage]);

  const handleStageChange = (nextIndex) => {
    if (nextIndex >= 0 && nextIndex <= 6) {
      setCurrentStage(nextIndex);
      if (nextIndex === 3) playTraditionalTone('mystic');
      else if (nextIndex === 6) playTraditionalTone('ending');
      else playTraditionalTone('click');
    }
  };

  const handleSoundToggle = () => {
    const nextMute = !isMuted;
    setIsMuted(nextMute);
    if (nextMute) {
      if ('speechSynthesis' in window) {
        window.speechSynthesis.cancel();
      }
    } else {
      playTraditionalTone('click');
    }
  };

  const resetGame = () => {
    setCurrentStage(0);
    setSelectedEnding(null);
    setFogCleared(false);
    setFogPixels(Array(12).fill(true));
    setPuzzleConnected({ eulmildae: null, daedongriver: null, bubyeokjeong: null });
    setPuzzleSolved(false);
    playTraditionalTone('success');
    speakVoice(narrationVoices.stage0);
  };

  const handleClearFogBlock = (index) => {
    if (fogCleared) return;
    const nextFog = [...fogPixels];
    nextFog[index] = false;
    setFogPixels(nextFog);
    playTraditionalTone('click');
    
    const clearedCount = nextFog.filter(item => !item).length;
    if (clearedCount >= 9) {
      setFogCleared(true);
      playTraditionalTone('success');
    }
  };

  const memoryPieces = [
    { id: 'mem1', speaker: "동명왕", quote: "웅혼한 고구려의 건국 기상이 을밀대 성벽 아래 굽이치며 대동강을 굽어보는도다.", target: 'eulmildae' },
    { id: 'mem2', speaker: "기녀 홍장", quote: "푸른 물결 위에 연인의 실루엣을 띄우며, 홀로 가녀린 슬픔의 구름을 노래하노라.", target: 'daedongriver' },
    { id: 'mem3', speaker: "방랑자 시습", quote: "부벽정 누각에 홀로 기대어 흥하고 쇠하는 천년의 사직을 헤아리니 참으로 허무하도다.", target: 'bubyeokjeong' }
  ];

  const handlePlacePiece = (targetSlot, pieceId) => {
    if (puzzleSolved) return;
    const piece = memoryPieces.find(p => p.id === pieceId);
    if (piece && piece.target === targetSlot) {
      const nextConnected = { ...puzzleConnected, [targetSlot]: pieceId };
      setPuzzleConnected(nextConnected);
      setSelectedPiece(null);
      playTraditionalTone('click');

      if (Object.values(nextConnected).filter(Boolean).length === 3) {
        setPuzzleSolved(true);
        playTraditionalTone('success');
      }
    }
  };

  return (
    <div className="min-h-screen bg-[#FDFBF7] text-stone-850 font-serif flex flex-col selection:bg-amber-100">
      
      {/* 우아한 한국 전통 매화단 채색 장식선 */}
      <div className="h-2 bg-gradient-to-r from-amber-200 via-orange-100 to-yellow-200 w-full shadow-sm" />

      {/* 헤더 */}
      <header className="border-b border-amber-100/60 bg-[#FAF7F0]/90 backdrop-blur-md sticky top-0 z-40 px-6 py-4">
        <div className="max-w-4xl mx-auto flex justify-between items-center">
          
          <div className="flex items-center gap-3">
            <div className="p-2 bg-amber-100/50 rounded-xl flex items-center justify-center shadow-sm">
              <Compass className="w-5 h-5 text-amber-800" />
            </div>
            <div>
              <h1 className="text-base md:text-lg font-bold text-stone-900 tracking-tight flex items-center gap-2">
                사유록 (四遊錄): 금오신화 가상 유람
              </h1>
            </div>
          </div>

          <div className="flex items-center gap-3">
            <button 
              onClick={resetGame} 
              className="p-2 hover:bg-amber-100/50 text-stone-600 transition-all bg-white border border-amber-100 rounded-lg shadow-xs" 
              title="첫 여정으로"
            >
              <RotateCcw className="w-4 h-4" />
            </button>
            <button 
              onClick={handleSoundToggle} 
              className="p-2 hover:bg-amber-100/50 text-stone-600 transition-all bg-white border border-amber-100 rounded-lg flex items-center gap-1.5 shadow-xs" 
              title={isMuted ? "나레이션 켜기" : "나레이션 끄기"}
            >
              {isMuted ? <VolumeX className="w-4 h-4 text-stone-400" /> : <Volume2 className="w-4 h-4 text-amber-700" />}
              <span className="text-xs font-sans text-stone-500 font-semibold">{isMuted ? "음성 꺼짐" : "음성 재생중"}</span>
            </button>
          </div>

        </div>
      </header>

      {/* 메인 뷰포트 레이아웃 */}
      <main className="flex-1 max-w-4xl w-full mx-auto p-4 md:p-6 flex flex-col justify-center">
        
        {/* 중앙: 한지 서화 스타일 프레임 극장 */}
        <div className="bg-white border-2 border-amber-100/80 rounded-3xl overflow-hidden shadow-xl flex flex-col min-h-[520px]">
          
          {/* 수묵 채색화 느낌의 밝고 우아한 동양 무대 배경 */}
          <div className="flex-1 bg-gradient-to-b from-[#FAF6EE] via-amber-50/20 to-[#FAF6EE] relative p-6 flex flex-col justify-between overflow-hidden transition-all duration-700">
            
            {/* 전통 구름 레이어 배경 장식 */}
            <div className="absolute inset-0 pointer-events-none opacity-25">
              <svg className="w-full h-full text-amber-600/30" xmlns="http://www.w3.org/2000/svg">
                <defs>
                  <pattern id="korean-cloud" width="120" height="120" patternUnits="userSpaceOnUse">
                    <path d="M 0,60 Q 30,30 60,60 T 120,60" fill="none" stroke="currentColor" strokeWidth="0.5" />
                    <path d="M 10,70 Q 40,45 70,70 T 130,70" fill="none" stroke="currentColor" strokeWidth="0.25" />
                  </pattern>
                </defs>
                <rect width="100%" height="100%" fill="url(#korean-cloud)" />
              </svg>
            </div>

            {/* STAGE 0: 지도 투영 */}
            {currentStage === 0 && (
              <div className="z-10 my-auto max-w-xl mx-auto text-center flex flex-col items-center animate-fade-in py-8">
                
                {/* 동양풍 안개 구름 효과 내 지도 */}
                <div className="relative w-48 h-32 mb-8 border border-amber-200/50 rounded-2xl bg-[#FCFBF7] overflow-hidden flex items-center justify-center p-3 shadow-sm">
                  <div className="absolute inset-0 bg-gradient-to-br from-amber-100/10 to-transparent" />
                  <div className="w-full h-full border border-amber-200/30 rounded-lg relative flex flex-col justify-between p-2">
                    <span className="text-[9px] text-amber-800/60 self-start">조선 팔도 고지도</span>
                    <div className="grid grid-cols-3 gap-2 text-[10px] text-stone-500 font-bold">
                      <span>平壤</span> <span>開城</span> <span>南原</span>
                    </div>
                    <div className="absolute inset-x-0 bottom-1 h-6 opacity-30 bg-gradient-to-t from-amber-100 to-transparent" />
                  </div>
                  <div className="absolute text-[11px] text-amber-900 bg-[#FCFBF7] border border-amber-200 px-4 py-1 rounded-full font-bold shadow-md animate-bounce">
                    지도 피어오르다
                  </div>
                </div>

                <h3 className="text-2xl font-bold text-stone-900 mb-3">
                  기록되지 않은 공간은 사라진다
                </h3>
                <p className="text-xs text-stone-600 leading-relaxed max-w-sm font-sans">
                  평평한 옛날 종이 지도 위에 잠들어 있던 조선의 산과 강이 입체적으로 고요히 숨 쉬는 공간으로 부드럽게 일어섭니다.
                </p>

                <button
                  onClick={() => handleStageChange(1)}
                  className="mt-8 bg-gradient-to-r from-amber-600 to-orange-500 text-white font-semibold text-xs px-8 py-3 rounded-xl transition hover:scale-105 inline-flex items-center gap-2 shadow-lg shadow-amber-700/10"
                >
                  지도로 유람하기
                  <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}

            {/* STAGE 1: 거점 활성화 */}
            {currentStage === 1 && (
              <div className="z-10 my-auto max-w-lg mx-auto text-center flex flex-col items-center animate-fade-in py-4">
                
                <div className="w-full bg-[#FCFBF7]/90 border border-amber-100/80 p-6 rounded-2xl mb-6 shadow-sm">
                  <h4 className="text-xs text-amber-800 font-bold tracking-widest mb-4">현실의 공간과 거점의 빛</h4>
                  
                  <div className="flex flex-col gap-2 text-xs text-stone-700 text-left">
                    <div className="h-10 bg-amber-50/40 border border-amber-100/40 flex items-center justify-between px-4 rounded-xl">
                      <span>거점 하나: 남원 기린산 만복사지</span> <span className="text-amber-800 font-bold">봉화 밝혀짐</span>
                    </div>
                    <div className="h-10 bg-orange-50/30 border border-orange-100/40 flex items-center justify-between px-4 rounded-xl">
                      <span>거점 둘: 개성 덕제동 계곡</span> <span className="text-amber-800 font-bold">봉화 밝혀짐</span>
                    </div>
                    <div className="h-10 bg-amber-50/40 border border-amber-100/40 flex items-center justify-between px-4 rounded-xl">
                      <span>거점 셋: 평양 부벽정 대동강</span> <span className="text-amber-800 font-bold">봉화 밝혀짐</span>
                    </div>
                  </div>

                  <div className="flex justify-center gap-10 mt-6">
                    <div className="flex flex-col items-center">
                      <span className="w-2 h-2 rounded-full bg-orange-400 relative" />
                      <span className="text-[10px] text-stone-600 mt-2">평양</span>
                    </div>
                    <div className="flex flex-col items-center">
                      <span className="w-2 h-2 rounded-full bg-amber-400 relative" />
                      <span className="text-[10px] text-stone-600 mt-2">개성</span>
                    </div>
                    <div className="flex flex-col items-center">
                      <span className="w-2 h-2 rounded-full bg-yellow-500 relative" />
                      <span className="text-[10px] text-stone-600 mt-2">남원</span>
                    </div>
                  </div>
                </div>

                <p className="text-xs text-stone-500 max-w-sm font-sans mb-4">
                  은빛 한반도 영토 위에 기이한 세 유적지가 찬란하게 겹쳐졌습니다.
                </p>

                <button
                  onClick={() => handleStageChange(2)}
                  className="bg-amber-700 hover:bg-amber-600 text-white text-xs px-6 py-2.5 rounded-xl transition hover:scale-105 inline-flex items-center gap-2 shadow-sm"
                >
                  기린산 산속으로 향하기
                  <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}

            {/* STAGE 2: 기린산 안개 걷기 */}
            {currentStage === 2 && (
              <div className="z-10 my-auto max-w-xl mx-auto text-center flex flex-col items-center animate-fade-in w-full py-4">
                
                <h3 className="text-lg font-bold text-stone-900 mb-2">기린산의 아련한 원망의 구름</h3>
                <p className="text-xs text-stone-600 mb-6 font-sans">
                  안개 타일을 마우스로 클릭하여 맑게 지워내며, 그 아래에 깃든 만복사지의 흔적을 복원하십시오.
                </p>

                <div className="relative w-48 h-48 border border-amber-100 rounded-2xl overflow-hidden bg-white flex items-center justify-center shadow-md">
                  
                  {/* 드러나는 만복사터 */}
                  <div className="absolute inset-0 flex flex-col items-center justify-center text-center p-4 bg-[#FCFBF7]">
                    <BookMarked className="w-8 h-8 text-amber-700 mb-2" />
                    <h5 className="font-bold text-amber-900 text-sm">만복사지 (萬福寺址)</h5>
                    <p className="text-[10px] text-stone-600 leading-relaxed max-w-[140px] mt-1">
                      이별한 양생과 귀녀가 주사위 놀이로 영원한 동행을 약속했던 신비로운 절터.
                    </p>
                  </div>

                  {/* 안개 타일 격자 */}
                  {!fogCleared && (
                    <div className="absolute inset-0 grid grid-cols-3 gap-0.5 p-0.5 bg-[#FAF6EE]/75 backdrop-blur-[1.5px]">
                      {fogPixels.map((isActive, idx) => (
                        <button
                          key={idx}
                          onClick={() => handleClearFogBlock(idx)}
                          className={`rounded-lg transition-all duration-300 relative ${
                            isActive 
                              ? 'bg-white/95 hover:bg-white/30 border border-amber-100 shadow-xs' 
                              : 'bg-transparent pointer-events-none'
                          }`}
                        />
                      ))}
                    </div>
                  )}

                  {/* 안개 제거 완정 연출 */}
                  {fogCleared && (
                    <div className="absolute inset-0 bg-[#FCFBF7] flex flex-col items-center justify-center text-center p-4 animate-fade-in">
                      <CheckCircle2 className="w-8 h-8 text-amber-700 mb-1" />
                      <span className="text-xs font-bold text-amber-950">안개가 걷히고 만복사지 복원됨</span>
                    </div>
                  )}

                </div>

                <button
                  onClick={() => handleStageChange(3)}
                  disabled={!fogCleared}
                  className={`mt-6 text-xs px-6 py-2.5 rounded-xl transition-all inline-flex items-center gap-2 ${
                    fogCleared 
                      ? 'bg-amber-700 text-white hover:scale-105 shadow-md shadow-amber-700/10' 
                      : 'bg-stone-200 text-stone-400 cursor-not-allowed'
                  }`}
                >
                  갈림길의 선택지로 향하기
                  <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}

            {/* STAGE 3: 만복사 삼교 분기 */}
            {currentStage === 3 && (
              <div className="z-10 my-auto max-w-2xl w-full mx-auto text-center flex flex-col items-center animate-fade-in py-2">
                
                {!selectedEnding ? (
                  <>
                    <h3 className="text-lg font-bold text-stone-900 mb-2">지리산 깊은 자락, 당신의 선택</h3>
                    <p className="text-xs text-stone-600 mb-6 font-sans">
                      떠나는 고결한 귀녀를 바라보는 양생. 그의 고뇌 앞에서 당신은 어떤 가치를 가슴에 품으시겠습니까?
                    </p>

                    {/* 삼교 카드 목록 */}
                    <div className="grid grid-cols-1 md:grid-cols-3 gap-4 w-full text-left font-sans">
                      
                      {/* 유교 */}
                      <button
                        onClick={() => setSelectedEnding('confucian')}
                        className="bg-white hover:bg-amber-50/20 border border-amber-200 p-4 rounded-xl transition duration-300 flex flex-col justify-between shadow-xs"
                      >
                        <div>
                          <div className="flex items-center gap-2 mb-2">
                            <span className="w-2 h-2 rounded-full bg-blue-500" />
                            <h5 className="font-bold text-blue-900 text-xs">유교적 절의</h5>
                          </div>
                          <p className="text-[11px] text-stone-600 leading-relaxed">
                            평생 다른 사랑을 하지 않고 양생 홀로 정절과 절개를 수호하며 지조를 지킵니다.
                          </p>
                        </div>
                        <span className="text-[10px] text-blue-800 mt-4 block font-mono border-t border-stone-100 pt-2">
                          청색(靑色) 갈림길
                        </span>
                      </button>

                      {/* 불교 */}
                      <button
                        onClick={() => setSelectedEnding('buddhist')}
                        className="bg-white hover:bg-amber-50/20 border border-amber-200 p-4 rounded-xl transition duration-300 flex flex-col justify-between shadow-xs"
                      >
                        <div>
                          <div className="flex items-center gap-2 mb-2">
                            <span className="w-2 h-2 rounded-full bg-yellow-500" />
                            <h5 className="font-bold text-yellow-900 text-xs">불교적 초탈</h5>
                          </div>
                          <p className="text-[11px] text-stone-600 leading-relaxed">
                            현실의 상처를 벗어나 이승을 떠나 타국의 남자로 환생하는 귀녀와 고요한 윤회를 나눕니다.
                          </p>
                        </div>
                        <span className="text-[10px] text-yellow-800 mt-4 block font-mono border-t border-stone-100 pt-2">
                          황색(黃色) 갈림길
                        </span>
                      </button>

                      {/* 도교 */}
                      <button
                        onClick={() => setSelectedEnding('taoist')}
                        className="bg-white hover:bg-amber-50/20 border border-amber-200 p-4 rounded-xl transition duration-300 flex flex-col justify-between shadow-xs"
                      >
                        <div>
                          <div className="flex items-center gap-2 mb-2">
                            <span className="w-2 h-2 rounded-full bg-teal-500" />
                            <h5 className="font-bold text-teal-900 text-xs">도교적 신선</h5>
                          </div>
                          <p className="text-[11px] text-stone-600 leading-relaxed">
                            신령한 지리산 골짜기로 스며들어 평화롭게 약초를 캐며 신선이 되는 길을 택합니다.
                          </p>
                        </div>
                        <span className="text-[10px] text-teal-800 mt-4 block font-mono border-t border-stone-100 pt-2">
                          백색(白色) 갈림길
                        </span>
                      </button>

                    </div>
                  </>
                ) : (
                  <div className="animate-fade-in w-full">
                    <div className="bg-white border border-amber-200 p-6 rounded-2xl text-left max-w-md mx-auto shadow-sm">
                      
                      {selectedEnding === 'confucian' && (
                        <div>
                          <h4 className="text-sm font-bold text-blue-900 mb-2">지조와 정절의 수절</h4>
                          <p className="text-xs text-stone-600 leading-relaxed italic">
                            "양생은 만복사터에 홀로 외롭게 무릎 꿇어 은그릇을 껴안습니다. 평생 귀녀를 그리워하며 지조를 다하는 조선 선비의 기개가 수묵의 밤빛 위로 고결하게 흐릅니다."
                          </p>
                        </div>
                      )}

                      {selectedEnding === 'buddhist' && (
                        <div>
                          <h4 className="text-sm font-bold text-yellow-900 mb-2">초탈과 환생의 수레바퀴</h4>
                          <p className="text-xs text-stone-600 leading-relaxed italic">
                            "부질없는 현실의 은그릇과 이승의 무거운 끈들을 모두 내려놓습니다. 타국의 남자로 평온히 몸을 바꾸어 환생하는 귀녀의 신비한 은빛 실루엣을 아련히 응시합니다."
                          </p>
                        </div>
                      )}

                      {selectedEnding === 'taoist' && (
                        <div>
                          <h4 className="text-sm font-bold text-teal-900 mb-2">지리산 청학동의 소박한 도인</h4>
                          <p className="text-xs text-stone-600 leading-relaxed italic">
                            "인연의 질서마저 아득하게 흩어지는 안개 속으로 걸어 들어갑니다. 영산의 품속에서 은근하게 약초를 캐며 구름과 흐르는 소나무 바람 사이로 영생하는 신선이 되어 녹아듭니다."
                          </p>
                        </div>
                      )}

                      <div className="flex gap-2 justify-end mt-6 pt-4 border-t border-stone-100 font-sans">
                        <button
                          onClick={() => setSelectedEnding(null)}
                          className="text-[10px] bg-stone-100 hover:bg-stone-200 border border-stone-200 text-stone-700 font-bold px-3 py-1.5 rounded-lg transition"
                        >
                          다른 가치 선택
                        </button>
                        <button
                          onClick={() => handleStageChange(4)}
                          className="text-[10px] bg-amber-700 hover:bg-amber-600 text-white font-bold px-4 py-1.5 rounded-lg transition inline-flex items-center gap-1 shadow-xs"
                        >
                          부벽정으로 나아가기
                          <ArrowRight className="w-3.5 h-3.5" />
                        </button>
                      </div>

                    </div>
                  </div>
                )}

              </div>
            )}

            {/* STAGE 4: 평양 부벽정 퍼즐 */}
            {currentStage === 4 && (
              <div className="z-10 my-auto max-w-2xl w-full mx-auto text-center flex flex-col items-center animate-fade-in py-2">
                
                <h3 className="text-lg font-bold text-stone-900 mb-1">부벽정 대동강의 기억 복원</h3>
                <p className="text-xs text-stone-600 mb-6 font-sans">
                  과거 대동강가에 머물렀던 인물의 기억 조각을 알맞은 평양의 역사 공간에 동조하여 맞추십시오.
                </p>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4 w-full">
                  
                  {/* 왼쪽: 역사 유적 슬롯 */}
                  <div className="flex flex-col gap-2 font-sans">
                    
                    {/* 을밀대 */}
                    <div 
                      onClick={() => selectedPiece && handlePlacePiece('eulmildae', selectedPiece)}
                      className={`p-3 rounded-xl border transition cursor-pointer flex flex-col justify-between min-h-[64px] text-left ${
                        puzzleConnected.eulmildae 
                          ? 'bg-blue-50/50 border-blue-400 text-blue-900' 
                          : 'bg-white border-amber-150 hover:border-amber-300 text-stone-600'
                      }`}
                    >
                      <div className="flex justify-between items-center text-xs">
                        <span className="font-bold">을밀대 (乙密臺)</span>
                        <span className="text-[9px] font-serif text-stone-400">성벽 아래 기상</span>
                      </div>
                      <p className="text-[10px] mt-1 italic leading-tight text-stone-500">
                        {puzzleConnected.eulmildae 
                          ? "동명왕의 기상이 안착되었습니다."
                          : "기억을 이곳에 동조해 주십시오."}
                      </p>
                    </div>

                    {/* 대동강 */}
                    <div 
                      onClick={() => selectedPiece && handlePlacePiece('daedongriver', selectedPiece)}
                      className={`p-3 rounded-xl border transition cursor-pointer flex flex-col justify-between min-h-[64px] text-left ${
                        puzzleConnected.daedongriver 
                          ? 'bg-yellow-50/50 border-yellow-450 text-yellow-900' 
                          : 'bg-white border-amber-150 hover:border-amber-300 text-stone-600'
                      }`}
                    >
                      <div className="flex justify-between items-center text-xs">
                        <span className="font-bold">대동강 (大同江)</span>
                        <span className="text-[9px] font-serif text-stone-400">푸른 물결</span>
                      </div>
                      <p className="text-[10px] mt-1 italic leading-tight text-stone-500">
                        {puzzleConnected.daedongriver 
                          ? "홍장의 애달픈 슬픔이 깃들었습니다."
                          : "기억을 이곳에 동조해 주십시오."}
                      </p>
                    </div>

                    {/* 부벽정 */}
                    <div 
                      onClick={() => selectedPiece && handlePlacePiece('bubyeokjeong', selectedPiece)}
                      className={`p-3 rounded-xl border transition cursor-pointer flex flex-col justify-between min-h-[64px] text-left ${
                        puzzleConnected.bubyeokjeong 
                          ? 'bg-amber-50 border-amber-400 text-amber-900' 
                          : 'bg-white border-amber-150 hover:border-amber-300 text-stone-600'
                      }`}
                    >
                      <div className="flex justify-between items-center text-xs">
                        <span className="font-bold">부벽정 (浮碧亭) 누각</span>
                        <span className="text-[9px] font-serif text-stone-400">시습의 사색</span>
                      </div>
                      <p className="text-[10px] mt-1 italic leading-tight text-stone-500">
                        {puzzleConnected.bubyeokjeong 
                          ? "방랑자 시습의 고뇌가 묶였습니다."
                          : "기억을 이곳에 동조해 주십시오."}
                      </p>
                    </div>

                  </div>

                  {/* 오른쪽: 안착시킬 기억 조각 */}
                  <div className="flex flex-col gap-2 bg-white border border-amber-100 p-4 rounded-xl text-left shadow-xs">
                    <span className="text-[10px] text-stone-400 block mb-1 font-sans">부유하는 기억 조각 클릭</span>
                    
                    {memoryPieces.map((piece) => {
                      const isPlaced = Object.values(puzzleConnected).includes(piece.id);
                      const isSelected = selectedPiece === piece.id;

                      return (
                        <div
                          key={piece.id}
                          onClick={() => !isPlaced && setSelectedPiece(piece.id)}
                          className={`p-2.5 rounded-lg border text-xs transition-all ${
                            isPlaced 
                              ? 'opacity-30 bg-stone-50 border-stone-200 cursor-not-allowed' 
                              : isSelected
                                ? 'bg-amber-100/50 border-amber-500 translate-x-1 cursor-pointer font-bold'
                                : 'bg-[#FAF6EE]/50 border-amber-100/60 hover:border-amber-300 cursor-pointer'
                          }`}
                        >
                          <div className="flex justify-between items-center mb-1">
                            <span className="font-bold text-stone-800">{piece.speaker}</span>
                            {isPlaced && <span className="text-[8px] bg-emerald-100 text-emerald-850 px-1.5 py-0.2 rounded font-sans">안착됨</span>}
                          </div>
                          <p className="text-[10px] text-stone-600 italic leading-normal">
                            "{piece.quote}"
                          </p>
                        </div>
                      );
                    })}
                  </div>

                </div>

                <button
                  onClick={() => handleStageChange(5)}
                  disabled={!puzzleSolved}
                  className={`mt-6 text-xs px-6 py-2.5 rounded-xl transition-all inline-flex items-center gap-2 ${
                    puzzleSolved 
                      ? 'bg-amber-700 text-white hover:scale-105 shadow-md shadow-amber-700/10' 
                      : 'bg-stone-200 text-stone-400 cursor-not-allowed'
                  }`}
                >
                  사유록 집대성하기
                  <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}

            {/* STAGE 5: 사유록 집대성 */}
            {currentStage === 5 && (
              <div className="z-10 my-auto max-w-xl mx-auto text-center flex flex-col items-center animate-fade-in py-4">
                
                {/* 정갈한 서책 그래픽 */}
                <div className="relative w-32 h-44 bg-amber-800 rounded-lg p-1.5 shadow-lg border border-amber-900 transform -rotate-1 mb-6">
                  <div className="w-full h-full bg-[#FAF6EE] rounded border border-amber-200 p-3.5 flex flex-col justify-between text-stone-900">
                    <div className="text-center font-bold text-xs tracking-widest border-b border-stone-850 pb-1.5">
                      四遊錄 (사유록)
                    </div>
                    <div className="text-[9px] text-stone-700 text-left leading-relaxed mt-2">
                      <p>기린산 만복사지,</p>
                      <p>지리산 산봉우리,</p>
                      <p>평양 대동강 부벽정...</p>
                      <p className="mt-2 font-bold text-amber-950">당신만의 유람기.</p>
                    </div>
                    <div className="text-[8px] text-right text-stone-400">
                      김시습 저
                    </div>
                  </div>
                </div>

                <h4 className="text-lg font-bold text-stone-900 mb-2">공간을 걷고, 이야기를 한 권으로 엮다</h4>
                <p className="text-xs text-stone-600 leading-relaxed max-w-sm font-sans mb-6">
                  남원의 이별부터 평양의 서글픈 역사까지, 당신의 지혜가 닿아 만들어낸 방랑의 사색들이 목판 서책 《사유록》에 영구하게 결속됩니다.
                </p>

                <button
                  onClick={() => handleStageChange(6)}
                  className="bg-gradient-to-r from-amber-700 to-orange-500 text-white text-xs px-6 py-2.5 rounded-xl transition hover:scale-105 inline-flex items-center gap-2 shadow-lg shadow-amber-700/10"
                >
                  최종 유람 완수하기
                  <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}

            {/* STAGE 6: 종장 */}
            {currentStage === 6 && (
              <div className="z-10 my-auto max-w-xl mx-auto text-center flex flex-col items-center animate-fade-in py-4">
                
                <div className="w-10 h-10 rounded-full bg-amber-100 border border-amber-200 text-amber-800 flex items-center justify-center mb-3">
                  <Award className="w-5 h-5" />
                </div>

                <h4 className="text-lg font-bold text-amber-950 mb-1">
                  사유록: 금오신화 가상 유람 완수
                </h4>
                <p className="text-xs text-stone-500 font-mono tracking-widest uppercase mb-4">
                  SAYUROK CLASSICAL ODYSSEY COMPLETED
                </p>

                <div className="bg-white border border-amber-100/80 p-5 rounded-2xl text-left text-xs text-stone-600 leading-relaxed w-full max-w-xs shadow-xs mb-6">
                  <p className="font-bold text-amber-900 mb-2">🎓 가상 유람 완정 증서</p>
                  <p className="text-[11px] text-stone-500">
                    전쟁 속에 스러진 슬픈 연인들의 넋을 어루만지고, 지리산의 깊은 영성과 삼교 사상을 주체적인 여정 속에서 완수해 낸 찬란한 인문 유람을 기립니다.
                  </p>
                </div>

                <button
                  onClick={resetGame}
                  className="bg-stone-100 hover:bg-stone-200 text-stone-850 font-bold text-xs px-5 py-2 rounded-lg border border-stone-250 transition inline-flex items-center gap-1.5"
                >
                  <RotateCcw className="w-3.5 h-3.5" />
                  다시 처음으로 유람하기
                </button>
              </div>
            )}

          </div>

          {/* 하단 페이지 조작 컨트롤 */}
          <div className="bg-[#FAF7F0] px-6 py-3 border-t border-amber-100/60 flex justify-between items-center font-sans">
            <button
              onClick={() => handleStageChange(currentStage - 1)}
              disabled={currentStage === 0}
              className={`text-xs px-3 py-1.5 rounded-lg font-bold transition-all ${
                currentStage === 0 
                  ? 'text-stone-400 bg-stone-50 cursor-not-allowed' 
                  : 'text-stone-700 bg-white border border-amber-100 hover:bg-amber-50/50 shadow-xs'
              }`}
            >
              이전 장
            </button>
            
            <div className="flex gap-2.5">
              {Array.from({ length: 7 }).map((_, idx) => (
                <button
                  key={idx}
                  onClick={() => handleStageChange(idx)}
                  className={`w-2.5 h-2.5 rounded-full transition-all duration-300 border ${
                    idx === currentStage 
                      ? 'bg-amber-700 border-amber-800 scale-125' 
                      : 'bg-white border-amber-200 hover:bg-amber-100'
                  }`}
                />
              ))}
            </div>

            <button
              onClick={() => handleStageChange(currentStage + 1)}
              disabled={currentStage === 6}
              className={`text-xs px-3 py-1.5 rounded-lg font-bold transition-all ${
                currentStage === 6 
                  ? 'text-stone-400 bg-stone-50 cursor-not-allowed' 
                  : 'text-stone-700 bg-white border border-amber-100 hover:bg-amber-50/50 shadow-xs'
              }`}
            >
              다음 장
            </button>
          </div>

        </div>

      </main>

      {/* 풋터 */}
      <footer className="border-t border-amber-100/60 bg-[#FAF7F0]/40 py-6 text-center text-xs text-stone-400 mt-6 shadow-inner font-serif">
        <p>© 2026 Sayurok Interactive Engine. All classical texts and stories from Geumo Shinhwa.</p>
      </footer>

    </div>
  );
}# -
