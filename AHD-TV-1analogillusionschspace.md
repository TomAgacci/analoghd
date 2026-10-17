// ============================================================================
// AHD-TV-1 ANALOG ILLUSION ENGINE + CHANNEL SPACE
// Behavioral spec to implement on a small analog board (e.g., CRT Expansion PCB).
// This is NOT firmware; it’s “code as spec” for oscillators, dividers, RC envelopes,
// and transistor drivers.
// ============================================================================


// ---------------------------------------------------------------------------
// 1. MASTER CONSTANTS (TIMING ROOTS)
// ---------------------------------------------------------------------------

const double F_BURST_NTSC      = 3.579545e6;   // Hz, NTSC color burst crystal
const double F_FRAME_NTSC      = 29.97;        // Hz, classic NTSC frame rate
const double F_FIELD_NTSC      = 59.94;        // Hz, classic NTSC field rate
const double F_HORIZ_NTSC      = 15734.0;      // Hz, NTSC horizontal line rate

// High-frame-rate modes (conceptual)
const double F_FRAME_100       = 100.0;        // Hz, AHD-TV-1-100 mode
const double F_FRAME_1000      = 1000.0;       // Hz, AHD-TV-1-1000 mode


// ---------------------------------------------------------------------------
// 2. ILLUSION RESOLUTION CLASSES (VIRTUAL ANALOG GRID)
// ---------------------------------------------------------------------------

typedef enum {
    L0_LDTV,        // ~400x300 / 400x225
    L1_SD_480,      // ~640x480 / 640x360
    L2_HD_720,      // ~1280x960 / 1280x720
    L3_FULLHD_1080, // ~1920x1440 / 1920x1080
    L4_4K,          // ~3840x2880 / 3840x2160
    L5_8K,          // ~7680x5760 / 7680x4320
    L6_16K_24KLINE  // ~15360x11520 / 15360x8640 (virtual 24k-line-class)
} IllusionClass;


// ---------------------------------------------------------------------------
// 3. FRAME-RATE MODES
// ---------------------------------------------------------------------------

typedef enum {
    MODE_NTSC_30,   // 29.97 fps, 59.94 fields, full NTSC hertz
    MODE_AHD_100,   // 100 fps, high-frame-rate illusion
    MODE_AHD_1000   // ~1000 fps, hyper-smooth illusion
} FrameMode;


// ---------------------------------------------------------------------------
// 4. ILLUSION PROFILES (FIELD BEHAVIOR)
// ---------------------------------------------------------------------------

typedef enum {
    PROF_BREATHING_FIELD,   // HD Breathing Field (smooth luma sinusoid)
    PROF_CHROMATIC_DRIFT,   // HD Chromatic Drift (color cycling)
    PROF_GHOSTWAVE_BLINK,   // Ghostwave (blink-synced scene cuts)
    PROF_WHITEOUT,          // Full-field white flash
    PROF_STATIC_FIELD       // Constant field (baseline)
} IllusionProfile;


// ---------------------------------------------------------------------------
// 5. STATE STRUCTURES
// ---------------------------------------------------------------------------

typedef struct {
    FrameMode       mode;
    IllusionClass   illusionClass;
    IllusionProfile profile;
    double          f_frame;
    double          t_frame;
    double          f_field;
    double          f_horiz;
} AHDTV1Config;

typedef struct {
    unsigned long   frameIndex;
    unsigned long   fieldIndex;
    double          lumaLevel;      // 0.0..1.0
    double          chromaPhase;    // 0..1 (abstract color state)
    double          phaseOffset;    // seconds from frame start
    double          pulseWidth;     // seconds
} AHDTV1FrameState;


// ---------------------------------------------------------------------------
// 6. CONFIG INITIALIZATION
// ---------------------------------------------------------------------------

AHDTV1Config initConfig(FrameMode mode, IllusionClass cls, IllusionProfile prof) {
    AHDTV1Config cfg;
    cfg.mode          = mode;
    cfg.illusionClass = cls;
    cfg.profile       = prof;

    switch (mode) {
        case MODE_NTSC_30:
            cfg.f_frame = F_FRAME_NTSC;
            cfg.f_field = F_FIELD_NTSC;
            cfg.f_horiz = F_HORIZ_NTSC;
            break;

        case MODE_AHD_100:
            cfg.f_frame = F_FRAME_100;
            cfg.f_field = 2.0 * F_FRAME_100;
            cfg.f_horiz = F_HORIZ_NTSC;
            break;

        case MODE_AHD_1000:
            cfg.f_frame = F_FRAME_1000;
            cfg.f_field = 2.0 * F_FRAME_1000;
            cfg.f_horiz = F_HORIZ_NTSC;
            break;
    }

    cfg.t_frame = 1.0 / cfg.f_frame;
    return cfg;
}


// ---------------------------------------------------------------------------
// 7. ILLUSION PROFILE FUNCTIONS (ANALOG ENVELOPES)
// ---------------------------------------------------------------------------

double breathingLuma(unsigned long frameIndex, unsigned long cycleFrames) {
    double phase = (double)frameIndex / (double)cycleFrames;
    return 0.5 + 0.3 * sin(2.0 * M_PI * phase);
}

double chromaticDriftPhase(unsigned long fieldIndex) {
    unsigned long step = fieldIndex % 4; // 0..3
    return (double)step / 4.0;           // 0.0, 0.25, 0.5, 0.75
}

int ghostwaveGate(unsigned long frameIndex, unsigned long cycleFrames) {
    unsigned long pos = frameIndex % cycleFrames;
    if (pos < cycleFrames / 3)          return -1; // pre-blink
    else if (pos < 2 * cycleFrames / 3) return 0;  // mid-blink
    else                                return +1; // post-blink
}

double ghostwaveLuma(int gateState) {
    switch (gateState) {
        case -1: return 1.0;
        case  0: return 0.2;
        case +1: return 0.6;
        default: return 0.5;
    }
}


// ---------------------------------------------------------------------------
// 8. PHASE OFFSET AND PULSE WIDTH (VIRTUAL HORIZONTAL GRID)
// ---------------------------------------------------------------------------

double computePhaseOffset(const AHDTV1Config *cfg, unsigned long frameIndex) {
    double T_H = 1.0 / cfg->f_horiz;
    unsigned long k = frameIndex % 100;
    return (double)k * T_H;
}

double computePulseWidth(const AHDTV1Config *cfg, IllusionClass cls) {
    switch (cls) {
        case L0_LDTV:        return 0.5 * cfg->t_frame;
        case L1_SD_480:      return 0.4 * cfg->t_frame;
        case L2_HD_720:      return 0.3 * cfg->t_frame;
        case L3_FULLHD_1080: return 0.2 * cfg->t_frame;
        case L4_4K:          return 0.1 * cfg->t_frame;
        case L5_8K:          return 0.05 * cfg->t_frame;
        case L6_16K_24KLINE: return 0.02 * cfg->t_frame;
    }
    return 0.2 * cfg->t_frame;
}


// ---------------------------------------------------------------------------
// 9. FRAME UPDATE LOGIC
// ---------------------------------------------------------------------------

AHDTV1FrameState updateFrame(const AHDTV1Config *cfg,
                             unsigned long frameIndex,
                             unsigned long fieldIndex) {
    AHDTV1FrameState st;
    st.frameIndex = frameIndex;
    st.fieldIndex = fieldIndex;

    switch (cfg->profile) {
        case PROF_BREATHING_FIELD:
            st.lumaLevel   = breathingLuma(frameIndex, 60);
            st.chromaPhase = 0.0;
            break;

        case PROF_CHROMATIC_DRIFT:
            st.lumaLevel   = 0.7;
            st.chromaPhase = chromaticDriftPhase(fieldIndex);
            break;

        case PROF_GHOSTWAVE_BLINK: {
            int gate       = ghostwaveGate(frameIndex, 90);
            st.lumaLevel   = ghostwaveLuma(gate);
            st.chromaPhase = 0.0;
            break;
        }

        case PROF_WHITEOUT:
            st.lumaLevel   = 1.0;
            st.chromaPhase = 0.0;
            break;

        case PROF_STATIC_FIELD:
        default:
            st.lumaLevel   = 0.5;
            st.chromaPhase = 0.0;
            break;
    }

    st.phaseOffset = computePhaseOffset(cfg, frameIndex);
    st.pulseWidth  = computePulseWidth(cfg, cfg->illusionClass);

    return st;
}


// ---------------------------------------------------------------------------
// 10. PERCEPTUAL HD CERTIFICATION LOGIC
// ---------------------------------------------------------------------------

bool passesMotionSmoothness(const AHDTV1Config *cfg) {
    return (cfg->illusionClass >= L3_FULLHD_1080);
}

bool passesFieldUniformity(const AHDTV1Config *cfg) {
    return (cfg->illusionClass >= L2_HD_720);
}

bool passesColorStability(const AHDTV1Config *cfg) {
    return (cfg->illusionClass >= L3_FULLHD_1080);
}

bool passesGhostwaveCuts(const AHDTV1Config *cfg) {
    return (cfg->illusionClass >= L4_4K);
}

bool isPerceptualHD(const AHDTV1Config *cfg) {
    return passesMotionSmoothness(cfg)
        && passesFieldUniformity(cfg)
        && passesColorStability(cfg)
        && passesGhostwaveCuts(cfg);
}

bool isPerceptualUHD_24K(const AHDTV1Config *cfg) {
    return (cfg->illusionClass >= L5_8K) && isPerceptualHD(cfg);
}


// ---------------------------------------------------------------------------
// 11. CHANNEL / SUBCHANNEL SPACE
// ---------------------------------------------------------------------------

typedef struct {
    unsigned long   channelId;      // 0..999999
    unsigned int    subchannelId;   // 1..999
    FrameMode       mode;
    IllusionClass   illusionClass;
    IllusionProfile profile;
} AHDTV1Channel;

AHDTV1Channel makeChannel(unsigned long ch, unsigned int sub,
                          FrameMode mode,
                          IllusionClass cls,
                          IllusionProfile prof) {
    AHDTV1Channel c;
    c.channelId     = ch;
    c.subchannelId  = sub;
    c.mode          = mode;
    c.illusionClass = cls;
    c.profile       = prof;
    return c;
}

AHDTV1Config configFromChannel(const AHDTV1Channel *ch) {
    return initConfig(ch->mode, ch->illusionClass, ch->profile);
}


// ---------------------------------------------------------------------------
// 12. EXAMPLE CHANNELS
// ---------------------------------------------------------------------------

void exampleChannelMap() {
    AHDTV1Channel ch0_1 = makeChannel(
        0, 1,
        MODE_NTSC_30,
        L3_FULLHD_1080,
        PROF_BREATHING_FIELD
    );

    AHDTV1Channel ch10_100 = makeChannel(
        10, 100,
        MODE_AHD_100,
        L4_4K,
        PROF_CHROMATIC_DRIFT
    );

    AHDTV1Channel chMax = makeChannel(
        999999, 999,
        MODE_AHD_1000,
        L6_16K_24KLINE,
        PROF_GHOSTWAVE_BLINK
    );
}


// ---------------------------------------------------------------------------
// 13. MAIN LOOP (ABSTRACT ENGINE)
// ---------------------------------------------------------------------------

void runAHDTV1Engine(AHDTV1Config *cfg) {
    unsigned long frameIndex = 0;
    unsigned long fieldIndex = 0;

    while (true) {
        AHDTV1FrameState st = updateFrame(cfg, frameIndex, fieldIndex);

        // Hardware mapping on your CRT Expansion PCB:
        // - phaseOffset  -> delay network / gating from frame start
        // - pulseWidth   -> RC timing for dot ON duration
        // - lumaLevel    -> transistor current driver for LED/CRT brightness
        // - chromaPhase  -> LED color selection or symbolic “color mode”

        frameIndex++;
        fieldIndex += 2;
        // In analog, timing is enforced by oscillators/dividers, not sleep().
    }
}


// ============================================================================
// This is the full behavioral “code” for AHD-TV-1:
// - You implement it on that Thingiverse CRT Expansion PCB with:
//   - 1 crystal or RC master oscillator,
//   - a small divider chain,
//   - RC envelopes,
//   - a dot driver tied into the CRT or an LED.
// ============================================================================
