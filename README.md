# Biometric-Emotional MFA

A two-factor login. Factor one is a password. Factor two is a liveness-bound
biometric assertion from a local authenticator that reads your face and your
expression.


The whole point of the design is this: your biometric never leaves your machine.
The server only ever sees a signature. No face, no embedding, no template is
sent anywhere.

## How it works

Three parts talk to each other.

- `login.html` is the browser page. It takes the password and then polls for the
  second factor.
- `mfa_server.py` is the relying party. It checks the password, issues a
  one-time nonce and a random challenge, verifies the signature, and holds the
  policy.
- `authenticator.py` is a local daemon. It runs the liveness challenge, unlocks
  an Ed25519 private key with your face and PIN, signs the server's nonce, then
  forgets the key.

The flow for one login:

1. You submit username and password to the server.
2. The server checks the scrypt hash. If it passes, it issues a fresh nonce and
   a challenge sequence, valid for a few seconds.
3. The authenticator claims the pending login and runs the challenge at the
   camera. You make the expressions and head turns it asks for.
4. The face embeddings collected during the challenge, plus your PIN, unlock the
   sealed private key.
5. The authenticator signs the server's nonce and posts the assertion.
6. The server verifies the signature against your stored public key and lets you
   in.

What the server stores per user: a username, a scrypt hash of the password, and
your Ed25519 public key. That is all. It is in `mfa_users.json`.

## The biometric pipeline

- YuNet detects the face and five landmarks.
- `alignCrop` warps the face to a 112x112 crop using those landmarks.
- SFace turns the crop into a 128-dimension embedding.
- Cosine distance to your enrolled template decides identity. Below the
  calibrated threshold is you.
- A separate four-class CNN reads emotion: angry, happy, neutral, sad.

I started with LBPH and it came out at 30.1% EER. The reason was worse than the
number. It was separating on capture medium, not on identity. Aligned SFace came
out at 0.00% EER on the data I had. There is a caveat on that number further
down.

Emotion started in Keras at 864 ms per frame, which is about 1.2 fps and useless
for a live challenge. I exported it to ONNX and ran it through OpenCV's DNN
module. It dropped to 2.1 ms per frame, a 410x speedup, with a maximum output
difference of 3.4e-10 against the original.

## The cryptography

This is the graded part, so here is what each primitive does and why it is
there.

- **scrypt** hashes the password on the server.
- **Ed25519** signs the server's nonce. The server verifies with the stored
  public key. The private key lives only in memory, and only for as long as it
  takes to sign.
- **Challenge-response** keeps it fresh. The server issues a new nonce and a new
  random challenge each login. A captured transcript is stale the moment it is
  used.
- **HMAC-SHA256** derives the challenge and MACs the transcript.
- **Fuzzy commitment** (Juels and Wattenberg) binds a secret to your face. A face
  is never identical twice, so you cannot hash it like a password. Fuzzy
  commitment tolerates the small variation and still recovers one fixed secret.
- **Sign-random-projection** (a SimHash) turns the 128-dimension embedding into a
  bit string.
- **Reliable-bit selection** keeps only the bits that stay stable across your
  enrolment captures. This is what makes the template cancelable. Change the
  projection seed and you get a fresh, unlinkable template from the same face.
- **A repetition code** corrects the bits that still flip between captures.
- **HKDF-SHA256** derives the AES key from the recovered secret and the PIN.
- **AES-256-GCM** seals the Ed25519 private key. The GCM tag doubles as the
  check. A wrong face or a wrong PIN produces a wrong key, the tag fails to
  verify, and nothing unlocks. So the system never has to store anything that
  says "this is the right face". The absence of that stored check is the point.

Key binding in one line: your face and your PIN together unlock a private key
that is otherwise sealed, and the face itself is never stored to verify it.

## Running it

You need Python 3, OpenCV built with the contrib modules, the `cryptography`
package, and a webcam.

Two model files are not in the repo because they are large public weights.
Download both into `models/`:

- SFace: `face_recognition_sface_2021dec.onnx` from the OpenCV Zoo.
- YuNet: `face_detection_yunet_2023mar.onnx` from the OpenCV Zoo.

Then run the steps in order:

1. `Step3b_AlignedEnrolment.ipynb`. Set `USER` at the top to your name or handle,
   then it captures your face and builds your template. Your capture folders key
   off that name, so nothing is tied to one person.
2. `Step4b_PoseChallenge.ipynb` calibrates head pose. It saves the direction you
   calibrated on, so left reads as left.
3. `Step5_KeyBinding.ipynb` mints the keypair and builds the credential. Set
   `USER` to the same name you enrolled under.
4. `python mfa_server.py register --user <your-username> --credential models/credential.json`
5. `python mfa_server.py` starts the relying party.
6. `python authenticator.py` starts the daemon. It asks for your PIN once and
   holds it in memory until you stop it.
7. Open the login page through the server and type the username you registered.

Nothing in the code is tied to one person. Each person sets up on their own
machine: set `USER` in Step 3b and Step 5, enrol your own face, register under
your own name. The authenticator signs with whatever credential and template are
on that machine, and the server keys every account by username, so it already
holds more than one user.

If you forget your PIN, `python check_pin.py` tells you which one your credential
actually uses, by trying to unlock it with your own enrolment images.

## The notebooks

The step files are the full record of how the recognizer got built, from the
first broken version to the working one.

- **LBPH era**: `Step0`, `Step0c`, `Step1`, `Step1b`, `Step1c`, `Step1d`. The
  first recognizer and why it failed.
- **Moving to embeddings**: `Step1e` compares LBPH against deep embeddings.
  `Step1f` runs a leave-one-identity-out rotation.
- **Aligned SFace**: `Step2` is the live recognizer. `Step3a` is the alignment
  ablation. `Step3b` is enrolment.
- **Liveness**: `Step4` is the emotion challenge. `Step4b` adds head pose.
- **Crypto**: `Step5` is the key binding. `Step6` is the full MFA login.

Files with `_old` or `_second` suffixes are earlier iterations. I kept them for
history. The unsuffixed name is the one to use.

## Security notes and limitations


**The attestation gap.** Run `authenticator.py --simulate` and it signs a
fabricated transcript with your real credential, with no camera involved, and the
server accepts it. That is not a bug in the server. A signature proves you hold
the key. It does not prove a camera was in the loop. Closing that gap needs
hardware attestation this project does not have. The simulate flag exists to make
the gap visible, not to hide it.

**The PIN is four digits.** It is a second secret, not a strong one. Treat it as
one factor among several, never as a password.

**One live impostor.** I only had one other person to test false accepts
against. So the 0.00% EER has a wide confidence interval on the false-accept
side. The number is honest for the data. The data is thin, and the write-up says
so.

**Do not commit your biometrics or keys.** See `.gitignore`. These files must
never be pushed: `models/liveness_key.bin` (a raw key), `models/credential.json`
(your sealed key and the fuzzy-commitment helper), `models/template*.npy` (your
face as a vector), and the LBPH `*.yml` models (trained on your face). Pushing
any of them would break the one promise the project makes.
