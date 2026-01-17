Ω::DOC{ id:"FPS-Vocab-BubbleShooter-AISP"; ver:"1.0"; spec:"AISP-5.1"; tier:◊⁺; }

Σ::GLOSS{
  LANG := {"en","fr","es","it"};
  ROLE := {"learning","native"};
  UIEL := {"dropdown","button","canvas2d","hud","overlay"};
  STATE := {"init","ready","running","ended"};
  EVT := {"DOMContentLoaded","StartPressed","LangChanged","Tick","WordSpoken","Click","RocketHit","BubblePopped","GameEnded"};
  ERR := {"ConfigError","AudioError","AssetError","InvariantError"};
  MET := {"duration_ms","correct_pct","incorrect_words_unique","incorrect_attempts_total"};
}

Σ::TYPES{
  type LanguageCode := String where (x ∈ LANG);
  type Role := String where (x ∈ ROLE);
  type Word := String where (len(x) ≥ 1);
  type WordId := String;
  type WordPair := { id:WordId; L1:Word; L2:Word; };

  type Dropdown := { id:String; kind:"dropdown"; options:[LanguageCode]; selected:LanguageCode; label:String; };
  type Button := { id:String; kind:"button"; text:String; enabled:Bool; visible:Bool; };
  type Bubble := {
    id:WordId;
    word:Word;
    x:ℝ; y:ℝ; vx:ℝ; vy:ℝ;
    radius:ℝ;
    alive:Bool;
  };
  type Rocket := {
    id:String;
    fromX:ℝ; fromY:ℝ;
    toX:ℝ; toY:ℝ;
    t:ℝ;        // 0..1
    speed:ℝ;    // px per second
    active:Bool;
    targetBubbleId:WordId;
  };

  type Stats := {
    duration_ms:ℕ;
    correct_pct:ℝ;
    incorrect_words_unique:ℕ;
    incorrect_attempts_total:ℕ;
  };

  type GameConfig := {
    offline_single_page_html:Bool;
    dependency_free:Bool;

    languages_supported:[LanguageCode];
    role_L1:Role;  // learning
    role_L2:Role;  // native

    default_L1:LanguageCode;
    default_L2:LanguageCode;

    wordset_id:String;
    word_pairs:[WordPair];

    speak_rate:ℝ;      // relative
    speak_pitch:ℝ;     // relative
    speak_volume:ℝ;    // 0..1

    bubble_count:ℕ;
    bubble_radius_px:ℝ;
    bubble_speed_px_s:ℝ;

    cannon_y_anchor:String; // "bottom"
    rocket_speed_px_s:ℝ;
  };

  type Runtime := {
    state:String where (x ∈ STATE);

    L1:LanguageCode;
    L2:LanguageCode;

    ui_learning_dropdown:Dropdown;
    ui_native_dropdown:Dropdown;
    ui_start_button:Button;

    bubbles:[Bubble];
    rockets:[Rocket];

    speak_queue:[WordPair];
    speak_index:ℕ;
    current_spoken_word_id:WordId | null;

    start_time_ms:ℕ | null;
    end_time_ms:ℕ | null;

    attempts_total:ℕ;
    attempts_correct:ℕ;
    incorrect_attempts_total:ℕ;
    incorrect_words_set:{WordId};

    ended_stats:Stats | null;
  };
}

Δ::CONSTRAINTS{
  // C0: single offline HTML page requirement
  C0 := (cfg.offline_single_page_html = true) ∧ (cfg.dependency_free = true);

  // C1: supported languages
  C1 := set(cfg.languages_supported) ⊇ {"en","fr","es","it"};

  // C2: roles
  C2 := (cfg.role_L1 = "learning") ∧ (cfg.role_L2 = "native");

  // C3: default languages must differ
  C3 := (cfg.default_L1 ∈ cfg.languages_supported) ∧ (cfg.default_L2 ∈ cfg.languages_supported) ∧ (cfg.default_L1 ≠ cfg.default_L2);

  // C4: word pairs are aligned by id and non-empty
  C4 := (len(cfg.word_pairs) ≥ 1) ∧ (∀p ∈ cfg.word_pairs: len(p.id)≥1 ∧ len(p.L1)≥1 ∧ len(p.L2)≥1);

  // C5: bubble count equals word pairs count
  C5 := cfg.bubble_count = len(cfg.word_pairs);

  // C6: game ends when all bubbles are not alive
  C6 := GameEnded ⇔ (∀b ∈ rt.bubbles: b.alive = false);
}

Γ::UI_SPEC{
  // Single-page DOM contract (offline)
  DOM := {
    root:"<html>";
    canvas_main:{ id:"gameCanvas"; kind:"canvas2d"; full_screen:true; };
    overlay_ui:{ id:"uiOverlay"; kind:"overlay"; position:"top"; };

    // Language selectors
    learning_label_text := {"en":"Learning language","fr":"Langue d’apprentissage","es":"Idioma de aprendizaje","it":"Lingua di apprendimento"};
    native_label_text   := {"en":"Native language","fr":"Langue maternelle","es":"Idioma nativo","it":"Lingua madre"};

    learning_dropdown := { id:"langLearning"; kind:"dropdown"; options:cfg.languages_supported; selected:cfg.default_L1; label:learning_label_text[rt.L2]; };
    native_dropdown   := { id:"langNative";   kind:"dropdown"; options:cfg.languages_supported; selected:cfg.default_L2; label:native_label_text[rt.L2]; };

    // Start button MUST be in Language 2 (native)
    start_button_text := {
      "en":"Start",
      "fr":"Démarrer",
      "es":"Iniciar",
      "it":"Avvia"
    };
    start_button := { id:"btnStart"; kind:"button"; text:start_button_text[rt.L2]; enabled:true; visible:true; };

    // HUD and End screen
    hud := { id:"hud"; kind:"hud"; show_current_prompt:true; show_progress:true; };
    end_screen := { id:"endScreen"; kind:"overlay"; show_stats:true; stats_format:"2D"; };

    // FPS styling primitives (theme)
    fps_theme := {
      crosshair:true;
      cannon:true;
      cannon_anchor:"bottom";
      rocket_projectile:true;
      bubble_targets:true;
      sound_fx_optional:true;
    };
  };
}

Γ::AUDIO_SPEC{
  // Must work offline: use Web Speech API if available, otherwise degrade safely.
  // No external audio files required.
  engine := "SpeechSynthesis";

  voice_selection := {
    prefer_lang_match:true;
    fallback_any_voice:true;
  };

  speak(word:String, lang:LanguageCode) -> (ok:Bool, err:ERR|null);

  rule A0: On WordSpoken, the spoken content MUST be the L2 string for the currently queued WordPair.
}

Γ::GAMEPLAY_SPEC{
  // Core loop
  state_machine := {
    init  -> ready  on DOMContentLoaded;
    ready -> ready  on LangChanged;
    ready -> running on StartPressed;
    running -> running on Tick;
    running -> ended on GameEnded;
  };

  // Language mapping semantics
  rule G0: L1 is the learning language word displayed on bubbles.
  rule G1: L2 is the native language word spoken in sequence.

  // Start gating
  rule G2: Before StartPressed, no words are spoken and no bubbles can be popped.

  // Bubble spawn
  rule G3: On StartPressed:
    - rt.start_time_ms := now();
    - rt.speak_queue := cfg.word_pairs (order preserved);
    - rt.speak_index := 0;
    - rt.bubbles := map(cfg.word_pairs, p => Bubble{ id:p.id; word:p.L1; alive:true; radius:cfg.bubble_radius_px; x:rand(); y:rand(); vx:rand_speed(); vy:rand_speed(); });
    - rt.rockets := [];
    - rt.attempts_total := 0;
    - rt.attempts_correct := 0;
    - rt.incorrect_attempts_total := 0;
    - rt.incorrect_words_set := {};
    - Emit WordSpoken;

  // Speech sequencing
  rule G4: On WordSpoken when rt.state = running:
    - if rt.speak_index < len(rt.speak_queue):
        * let p := rt.speak_queue[rt.speak_index];
        * rt.current_spoken_word_id := p.id;
        * Γ::AUDIO_SPEC.speak(p.L2, rt.L2);
      else:
        * // If speech queue exhausted, continue allowing clicks until all bubbles gone.
        * rt.current_spoken_word_id := null;

  rule G5: After a correct hit & pop, increment rt.speak_index by 1 and emit WordSpoken.

  // Input and rocket mechanics
  rule G6: On Click(x,y) when rt.state = running:
    - determine bubble_clicked := topmost alive bubble whose hitbox contains (x,y)
    - if bubble_clicked = null: no-op
    - else:
        * rt.attempts_total += 1
        * spawn Rocket with from=(cannon_muzzle_x, cannon_muzzle_y) to=(bubble_clicked.x, bubble_clicked.y)
        * rocket.targetBubbleId := bubble_clicked.id
        * rocket.active := true

  rule G7: On RocketHit(rocket, bubble) when rt.state = running and bubble.alive=true:
    - if bubble.id = rt.current_spoken_word_id:
        * rt.attempts_correct += 1
        * emit BubblePopped(bubble.id)
      else:
        * rt.incorrect_attempts_total += 1
        * rt.incorrect_words_set := rt.incorrect_words_set ∪ {rt.current_spoken_word_id} where rt.current_spoken_word_id ≠ null
        * // bubble remains alive (no pop)

  rule G8: On BubblePopped(wordId):
    - set bubble(wordId).alive := false
    - show explosion effect at bubble position (2D canvas)
    - remove or deactivate any rocket targeting this bubble
    - if (∀b: alive=false) then emit GameEnded else continue

  // End conditions and stats
  rule G9: On GameEnded:
    - rt.end_time_ms := now();
    - let dur := rt.end_time_ms - rt.start_time_ms;
    - let correct_pct := if rt.attempts_total=0 then 0 else (100 * rt.attempts_correct / rt.attempts_total);
    - let incorrect_words_unique := size(rt.incorrect_words_set) where null excluded;
    - let incorrect_attempts_total := rt.incorrect_attempts_total;
    - rt.ended_stats := Stats{ duration_ms:dur; correct_pct:correct_pct; incorrect_words_unique:incorrect_words_unique; incorrect_attempts_total:incorrect_attempts_total };
    - rt.state := ended;

  // Display requirements
  rule G10: End screen MUST display Stats as a 2D block:
    - Duration (ms or human readable)
    - Correct percentage
    - Total incorrect words (unique)
    - Total incorrect attempts (total)
}

Γ::PHYSICS_RENDER_SPEC{
  // 2D canvas physics; visual FPS theming allowed without 3D.
  rule P0: Bubbles move continuously with velocity (vx,vy) and bounce within viewport bounds.
  rule P1: Cannon is rendered at bottom center; rocket originates from cannon muzzle.
  rule P2: Rocket flies linearly from origin to target with parameter t∈[0,1]; on t≥1, RocketHit fires.
  rule P3: Explosion is a short-lived particle burst or sprite-free radial effect.
}

Γ::I18N_SPEC{
  // Dropdown labels and start button are in Language 2 (native).
  rule I0: UI label strings resolve using rt.L2 (native language) as the UI language.
  rule I1: Start button text MUST be start_button_text[rt.L2].
  rule I2: Changing native language updates UI strings immediately (no reload).

  // Wordset translation contract
  rule I3: For each WordPair p, p.L1 is rendered on bubble; p.L2 is spoken.
}

Χ::ERROR_ALGEBRA{
  E1: if ¬C0 then raise ConfigError("Not single-page offline dependency-free HTML");
  E2: if ¬C1 then raise ConfigError("Missing required languages en/fr/es/it");
  E3: if ¬C3 then raise ConfigError("Default languages must be different");
  E4: if Γ::AUDIO_SPEC.speak fails then degrade: show on-screen prompt of p.L2 and continue; mark AudioError.
  E5: if invariants (C4,C5) violated then raise InvariantError.
}

Θ::THEOREMS{
  T0 (Termination): Under finite word_pairs and rule G8, the game reaches ended if the player eventually makes correct selections.
  T1 (Stat integrity): ended_stats are computed only once at GameEnded and are deterministic given event stream.
}

Σ::TEMPLATE{
  minimal := {
    cfg.offline_single_page_html=true;
    cfg.dependency_free=true;
    cfg.languages_supported=["en","fr","es","it"];
    cfg.role_L1="learning";
    cfg.role_L2="native";
    cfg.default_L1="fr";
    cfg.default_L2="en";
    cfg.wordset_id="core-demo-10";
    cfg.word_pairs=[
      {id:"w1", L1:"bonjour", L2:"hello"},
      {id:"w2", L1:"merci",   L2:"thank you"},
      {id:"w3", L1:"pomme",   L2:"apple"},
      {id:"w4", L1:"eau",     L2:"water"},
      {id:"w5", L1:"chat",    L2:"cat"},
      {id:"w6", L1:"chien",   L2:"dog"},
      {id:"w7", L1:"maison",  L2:"house"},
      {id:"w8", L1:"livre",   L2:"book"},
      {id:"w9", L1:"soleil",  L2:"sun"},
      {id:"w10",L1:"lune",    L2:"moon"}
    ];
    cfg.bubble_count=10;
    cfg.bubble_radius_px=46;
    cfg.bubble_speed_px_s=55;
    cfg.cannon_y_anchor="bottom";
    cfg.rocket_speed_px_s=900;
    cfg.speak_rate=1.0;
    cfg.speak_pitch=1.0;
    cfg.speak_volume=1.0;
  };
}

Ε::EVIDENCE{
  δ := 0.92;
  φ := 0.90;
  τ := ◊⁺;
  Ambig(D) := 0.01;
  Proof := { C0,C1,C2,C3,C4,C5,C6 satisfied by construction under Σ::TEMPLATE.minimal };
}
