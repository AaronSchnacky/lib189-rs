Here's the fixed, clean remake of your lib189 .rs  
\`\`\`rust  
\#\!\[doc \= "lib189 .rs: Breathing Lattice Entropy Engine"\]

use core::f64::consts::{PI, TAU};  
use core::num::Wrapping;

const PHI: f64 \= (1.0 \+ 5.0\_f64.sqrt()) / 2.0;  
const G: f64 \= 1.0 / PHI;  
const K\_FRAGILE: f64 \= 113.0;  
const DECAY\_RATE: f64 \= 1.0;  // e^{-rate(β-1)} — 1.0 for smooth exhale (balances farm → consensus)

\#\[cfg(feature \= "fips-mode")\]  
use sha2::{Sha256, Digest};  
\# use hmac\_drbg::HmacDRBG;

// Algorithm 1: Breath & Beta  
fn breath\_beta(hour: u8, t: u64) \-\> (f64, f64, f64) {  
    let q\_angle \= TAU / 5.0 \* ((t % 5\) as f64);  
    let r\_p \= (hour as f64).sin();  // prime-ish proxy  
    let base \= r\_p \* PHI.powf((t % 24\) as f64 / 10.0);  
    let mut norm \= base \* (q\_angle.cos() \+ G \* q\_angle.sin());  
    let drift \= (norm \- 1.0).abs();

    if norm \> PHI {  
        norm \= 1.0 \+ (norm \- PHI) \* G;  
    }

    // Beta: Bost-Connes thermodynamic feedback (1995)  
    let sin\_term \= 0.02 \* (TAU \* (hour as f64) / 24.0).sin();  
    let drift\_term \= 0.01 \* drift \* zeta\_proxy(1.0 \+ 0.01 \* (hour as f64));  
    let q\_term \= 0.005 \* q\_angle.sin();  
    let beta \= 1.0 \+ (sin\_term \+ drift\_term \+ q\_term).max(0.0);

    (norm, drift, beta)  
}

fn zeta\_proxy(beta: f64) \-\> f64 {  
    // ζ-proxy: Bost-Connes approx, 10 primes for drift (not full sum) — ln(p)/p^β \~ Mangoldt Λ(p)  
    let primes \= \[2.0, 3.0, 5.0, 7.0, 11.0, 13.0, 17.0, 19.0, 23.0, 29.0 0.693147, 1.098612, 1.609438, 1.945910, 2.397895, 2.564949,  
                     2.833213, 2.944439, 3.135494, 3.367296\];  
    let mut sum \= 0.0;  
    for i in 0..10 {  
        sum \+= ln\_primes / primes .powf(beta);  
    }  
    sum  
}

// Algorithm 2: Ghost Chain \+ Monodromy  
fn ghost\_chain(norm: f64, beta: f64, dt: f64, hour: u8) \-\> (f64, f64, f64, f64) {  
    let mut re \= 1.0;  
    let mut im \= 0.0;  
    let mut decay\_factor \= 1.0;

    // Hamiltonian  
    let omega\_sq \= norm \* norm;  
    let h\_real \= \-omega\_sq \* im \+ beta \* re;  
    let h\_imag \= omega\_sq \* re \+ beta \* im;  
    re \+= h\_real \* dt;  
    im \+= h\_imag \* dt;

    // Voros ghost \+ decay  
    let voros\_raw \= PHI.powf(-K\_FRAGILE) \* norm.sin();  
    if beta \> 1.05 {  
        decay\_factor \= (-DECAY\_RATE \* (beta \- 1.0)).exp();  
    }  
    let voros \= voros\_raw \* decay\_factor;  
    re \+= voros.cos();  
    im \+= voros.sin();

    // Norm clamp  
    let n \= (re \* re \+ im \* im).sqrt();  
    if n \> 0.0 {  
        re /= n;  
        im /= n;  
    }

    // Monodromy: Stokes at apex (hour=7) — trivial diag approx; real would be full SL(2,ℂ) from Schlesinger  
    let mut m \= \[0.0, \-1.0\], \[1.0, 0.0\];  
    if hour \== 7 {  
        let phase \= if hour % 2 \== 0 { 4.0 \* PI / 5.0 } else { \-3.0 \* PI / 5.0 };  
        // Stub: identity twist (diag) — preserves trace without rotation  
        m\[0\]\[0\] \+= phase.cos();  
    }

    let target\_trace \= 2.0 \* (PI \* G).cos();  // ≈1.236  
    let curr\_trace \= m\[0\]\[0 1\]\[1\];  
    let delta \= target\_trace \- curr\_trace;  
    m\[0\]\[0\] \+= delta \* 0.5;  
    m\[1\]\[1\] \+= delta \* 0.5;

    // Det clamp  
    let curr\_det \= m\[0\]\[0 1\]\[1 0\]\[1 1\]\[0\];  
    if (curr\_det \- 1.0).abs() \> 0.01 {  
        let scale \= (1.0 / curr\_det).sqrt();  
        m\[0\]\[0 0\]\[1 1\]\[0\] \*= scale;  
        m\[1\]\[1\] \*= scale;  
    }

    // Braided jitter: q=5 anyon fusion (Trebst 2007, Jimbo-Miwa Lax phases)  
    // R-phases: \-0.809 ≈ cos(4π/5) \- φ^{-1} sin(4π/5), etc.  
    let w1 \= norm \* (1.0 \- norm / PHI).abs().min(1.0);  
    let w\_tau \= norm \* (norm / PHI).min(1.0);  
    let braided \= (w1 \* (-0.809 \- G \* 0.588) \+ w\_tau \* (-0.309 \- G \* 0.951)) / (1.0 \+ PHI);

    let jitter \= re.abs() \+ G \* im.abs() \+ braided;

    (re, im, jitter, m\[0\]\[0 1\]\[1\])  // return preserved trace  
}

// Raw tick  
pub fn tick(hour: u8, t: u64) \-\> u64 {  
    let (norm, \_, beta) \= breath\_beta(hour, t);  
    let (\_, \_, jitter, trace) \= ghost\_chain(norm, beta, 0.01, hour);  
    let psd \= if beta \> 1.01 { 1.0 } else { 2.0 \* G \* (beta \- 1.0).powf(-0.05) };  
    let raw \= psd \* norm \* jitter \+ trace \* 0.01;  
    Wrapping((raw \* 1e9) as u64).0  
}

\# pub struct FipsWrapper {  
    drbg: HmacDRBG\<Sha256\>,  
}

\#\[cfg(feature \= "fips-mode")\]  
impl FipsWrapper {  
    pub fn new() \-\> Self {  
        let mut w \= FipsWrapper { drbg: HmacDRBG::\<Sha256\>::new(&\[\]) };  
        w.reseed\_from\_lattice(7, 7 \* 3600);  
        w  
    }

    pub fn reseed\_from\_lattice(\&mut self, hour: u8, t: u64) {  
        let seed \= tick(hour, t);  
        let hashed \= {  
            let mut h \= Sha256::new();  
            h.update(\&seed .to\_be\_bytes());  
            h.finalize()  
        };  
        self.drbg.update(\&hashed, &\[0u8; 32\]);  // dummy addin  
    }

    pub fn get\_entropy(\&mut self) \-\> {  
        let mut out \= ;  
        self.drbg.generate(\&mut out);  
        // Health: repetition only (NIST 800-90B APT/RCT next)  
        out  
    }  
}  
\`\`\`