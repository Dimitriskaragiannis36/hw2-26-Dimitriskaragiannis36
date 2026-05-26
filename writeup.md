**ΕΡΓΑΣΙΑ hw2-26-Dimitriskaragiannis36 ΣΤΟ ΜΑΘΗΜΑ HACK_INTRO**
*ΤΟΥ ΚΑΡΑΓΙΑΝΝΗ ΔΗΜΗΤΡΙΟΥ (1115202200293)*
# Writeup: wasm 3 (200 points)

**Η όλη φιλοσοφία της άσκησης**:
Η άσκηση αυτή περιστρέφεται γύρω από την αντίστροφη μηχανική (reverse engineering) ενός εκτελέσιμου αρχείου WebAssembly (wasm). Στόχος μας ήταν να κατανοήσουμε πώς λειτουργεί η συνάρτηση ελέγχου (checker) που εγκρίνει το flag και να βρούμε την κατάλληλη είσοδο που την ικανοποιεί. Με λίγα λόγια έπρεπε να "ξετυλίξουμε" έναν custom αλγόριθμο ωσότου πάρουμε πρόσβαση στο σημείο που ανάβει το πράσινο φως! 

**Βήμα 1ο: Αναγνώριση και πρώτη επαφή με το αρχείο**
Όταν πήραμε στα χέρια μας το αρχείο με όνομα `5245e7a6`, το πρώτο βήμα, όπως συνηθίζεται πάντα σε τέτοιες περιπτώσεις, ήταν να εξακριβώσουμε τι αρχείο είναι. Τρέχοντας την εντολή `file 5245e7a6` στο τερματικό, το σύστημα μας ενημέρωσε ξεκάθαρα:
`5245e7a6: WebAssembly (wasm) binary module version 0x1 (MVP)`

Δεδομένου ότι το WebAssembly σε binary μορφή δεν είναι δυνατόν να διαβαστεί από άνθρωπο, έπρεπε να το μετατρέψουμε σε πιο κατανοητή μορφή. Για τον σκοπό αυτό, χρησιμοποιήσαμε τα δημοφιλή εργαλεία της σουίτας WABT (WebAssembly Binary Toolkit). Συγκεκριμένα, τρέξαμε:
1. `wasm2wat 5245e7a6 -o 5245e7a6.wat` (για να εκμαιεύσουμε την textual μορφή)
2. `wasm-decompile 5245e7a6 -o 5245e7a6.c` (για να πάρουμε μια C-like αναπαράσταση, η οποία είναι πολύ πιο γνώριμη για ανάγνωση).

**Βήμα 2ο: Αναζητώντας τη "βελόνα στα άχυρα" (τον Checker)**
Ανοίγοντας το decompiled αρχείο (.c), αρχίσαμε να αναζητάμε τα λεγόμενα "exports", δηλαδή τις συναρτήσεις που το αρχείο μοιράζεται προς τα έξω για να κληθούν. Εκεί δεν άργησε να κάνει "μπαμ" μια συνάρτηση με το όνομα `export function check(a:int):int`. Αυτή η συνάρτηση παίρνει ως όρισμα μια διεύθυνση μνήμης `a` (ουσιαστικά τον δείκτη (pointer) προς το input/flag μας) και επιστρέφει έναν ακέραιο `int` (κατά πάσα πιθανότητα 1=επιτυχία, 0=αποτυχία). Ήταν σίγουρα ο στόχος μας.

Ξεκινήσαμε λοιπόν να ιχνηλατούμε τον κώδικα της `check`. Παρατηρήσαμε αμέσως πως δεν υπήρχε κάποιο απλό string comparison (π.χ. αν το input είναι "flag{auto}"). Αντιθέτως, η συνάρτηση:
1. Δέσμευε δυναμικά μια νέα περιοχή μνήμης μέσω μιας δικής της version της `malloc`/`memcpy` (της `f_ja`).
2. Αρχικοποιούσε κάποιους δείκτες και μετά έμπαινε σε έναν τεράστιο και "άβολο" βρόχο (loop) που χρησιμοποιούσε συνεχώς `br_table` (η wasm μορφή ενός τεράστιου switch-case που βλέπουμε στην C).

Αυτή η δομή (δηλαδή ένας ατελείωτος βρόχος με ένα switch-case στο κέντρο, το οποίο διαβάζει και διεκπεραιώνει εντολές) αποτελεί την **κλασική υπογραφή ενός custom Virtual Machine (VM)**! Συνειδητοποιήσαμε, δηλαδή, ότι ο δημιουργός της άσκησης έγραψε μέσα σε WebAssembly μια δική του μικρή εικονική μηχανή για να ελέγχει με αποκρυπτογραφημένο τρόπο το input μας.

**Βήμα 3ο: Αντίστροφη μηχανική της λογικής του VM**
Αφού καταλάβαμε τι γίνεται, επικεντρωθήκαμε στην κατανόηση της αρχιτεκτονικής του VM. Το μηχάνημα αυτό λειτουργεί πάνω σε μια νοητή "ταινία" (tape) 256 θέσεων και ένα δείκτη (ptr) που κινείται κατά μήκος της, θυμίζοντας έντονα λογικές τύπου Brainfuck ή θεωρητικών μηχανών Turing.
Αναλύοντας τι έκανε το κάθε "case" στο `br_table`, μπορέσαμε να χαρτογραφήσουμε (map) τις εντολές (opcodes) της μηχανής:

- `0` -> `HALT_OK` (εάν το πρόγραμμα φτάσει εδώ, ο κωδικός ήταν σωστός!)
- `1` -> `SET_CELL_IMM` (εκχώρηση μιας σταθερής τιμής στο τρέχον κελί της ταινίας)
- `2` -> `SET_PTR_IMM` (μετακίνηση του δείκτη σε ένα συγκεκριμένο κελί της ταινίας)
- `3` -> `READ_INPUT` (ανάγνωση του "επόμενου" χαρακτήρα του δικού μας flag)
- `4` -> `COPY_CELL` (αντιγραφή τιμής από άλλο κελί)
- `5` -> `INC_CELL` (αύξηση τιμής κελιού, κλασικό tape + 1)
- `6` -> `DEC_CELL` (μείωση τιμής)
- `7` -> `JZ` (άλμα (jump) στο πρόγραμμα, εφόσον το τρέχον κελί έχει τιμή 0)
- `8` -> `HALT_FAIL` (το input ήταν λάθος, διακοπή εκτέλεσης)

Επιστρέφοντας στον αποσυναρμολογημένο (ή λυμμένο όπως λένε στον στρατό) κώδικα, παρατηρήσαμε ότι η ρουτίνα φόρτωνε το πρόγραμμα του VM διαβάζοντας στατικά από το offset `1024` της κύριας μνήμης.

**Βήμα 4ο: Εξαγωγή του προγράμματος (Bytecode)**
Όσο καλά κι αν καταλαβαίναμε τη μηχανή, αυτό ήταν άχρηστο αν δεν βλέπαμε **τι** πρόγραμμα εκτελούσε. Έτσι, γράψαμε ένα Python script (περιλαμβάνεται ολόκληρο στο τέλος) για να διαβάσουμε το raw `.wasm` αρχείο, να ψάξουμε για το data section του wasm (που έχει πάντα `section_id = 11`), και να απομονώσουμε τα δεδομένα (bytes) που τοποθετούνται από τον compiler στο offset 1024. Διαβάζοντάς τα ως 32-bit ακέραιους (int32 array), βγάλαμε τον πραγματικό κώδικα (bytecode) του VM.
Χτίζοντας μια υποτυπώδη disassembler (που τύπωνε text όπως "SET_PTR_IMM 2" αντί για κωδικούς), κατορθώσαμε να δούμε όλη τη σειρά των εντολών. Είδαμε ξεκάθαρα πως το πρόγραμμα έκανε ακριβώς 20 πανομοιότυπους κύκλους ανάγνωσης-ελέγχου (`READ_INPUT`). Αυτό επιβεβαίωσε πως το input/flag μας πρέπει να είναι ***ακριβώς 20 χαρακτήρες*** (και κατά πάσα πιθανότητα χωρίς κανονικό padding τύπου `flag{...}`).

**Βήμα 5ο: Κάνοντας Brute-Force (το πολυπόθητο exploit flow)**
Τώρα είχαμε τα πάντα: Ξέραμε την αρχιτεκτονική του VM, καθως και τον κώδικα που εκτελεί. Το να καταλάβουμε τον μαθηματικό αλγόριθμο character-by-character μέσα στον πολύπλοκο αλγόριθμο ήταν χρονοβόρο. Υπήρχε όμως "ένα παράθυρο επιτυχίας"! 
Το custom VM διαβάζει έναν χαρακτήρα (`READ_INPUT`), κάνει μια σειρά από πράξεις, τον κρίνει, και:
- Εάν είναι ο λάθος χαρακτήρας εκτελεί άλμα και οδηγείται σε `HALT_FAIL`.
- Εάν είναι ο **σωστός** χαρακτήρας, περνάει επιτυχώς την λούπα και προχωράει ζητώντας να διαβάσει τον **επόμενο** χαρακτήρα μέσω του επόμενου `READ_INPUT`.

Με βάση αυτή τη λογική (ένα side-channel attack βασισμένο στο execution path), στήσαμε έναν **δικό μας emulator του VM σε Python**! Αντί να σπάσουμε τον αλγόριθμο θεωρητικά, δημιουργήσαμε έναν buffer 20 bytes (ας πούμε `['A', 'A', 'A', ...]`).
Για κάθε κελί `i` (από 0 μέχρι 19):
- Δοκιμάζαμε όλες τις δυνατές ascii τιμές (από το 0 έως το 255).
- Ρίχναμε τον buffer στον Python emulator μας.
- Μετρούσαμε πόσες φορές το VM εκτέλεσε την εντολή `READ_INPUT`. Αν έκανε πάνω από `i+1` reads, αυτό σήμαινε πως το γράμμα που μόλις δοκιμάσαμε ήταν το *σωστό* γιατί το VM ζήτησε με επιτυχία το επόμενο!
- Για τη θέση του 20ου χαρακτήρα, περιμέναμε ο emulator να μας επιστρέψει απλά το `HALT_OK`. 

**Βήμα 6ο: Εκτέλεση και Λύση**
Όλα ήταν έτοιμα, το Python VM έτρεχε. Το script άρχισε να "ξεκλειδώνει" γραμμικά τον κάθε χαρακτήρα ακριβώς επειδή το "oracle" του `READ_INPUT` λειτουργούσε τέλεια. Στην οθόνη του τερματικού μας τυπώθηκαν σταδιακά τα εξής αποτελέσματα:
> pos 0 byte 118 char v
> pos 1 byte 109 char m
> pos 2 byte 53 char 5
> pos 3 byte 95 char _
... και το τελευταίο "byte 110 char n" έφερε το περιζήτητο `HALT_OK`.

### Το Πλήρες Script (solve.py)
Το script που δημιουργήθηκε (βρίσκεται και στο αρχείο `solve.py`) κάνει αθόρυβα και αυτοματοποιημένα όλα τα παραπάνω: διαβάζει το wasm, φορτώνει τη μνήμη, μιμείται το VM και κάνει το brute-force του flag!

```python
#!/usr/bin/env python3
import struct
from pathlib import Path

OP_HALT_OK = 0
OP_SET_CELL_IMM = 1
OP_SET_PTR_IMM = 2
OP_READ_INPUT = 3
OP_COPY_CELL = 4
OP_INC_CELL = 5
OP_DEC_CELL = 6
OP_JZ = 7
OP_HALT_FAIL = 8

def read_varuint(data: bytes, pos: int) -> tuple[int, int]:
    result = 0
    shift = 0
    while True:
        b = data[pos]
        pos += 1
        result |= (b & 0x7f) << shift
        if b & 0x80 == 0:
            break
        shift += 7
    return result, pos

def load_program(wasm_path: Path, base: int = 1024, count: int = 1100) -> list[int]:
    wasm = wasm_path.read_bytes()
    pos = 8
    segments = []
    while pos < len(wasm):
        section_id = wasm[pos]
        pos += 1
        size, pos = read_varuint(wasm, pos)
        payload = wasm[pos : pos + size]
        pos += size
        if section_id != 11:  # Μόνο Data section προσπελαύνεται
            continue

        p = 0
        seg_count, p = read_varuint(payload, p)
        for _ in range(seg_count):
            flag, p = read_varuint(payload, p)
            # Ανάλογα την έκδοση flag προχωράμε το parsing του offset
            if flag == 0:
                if payload[p] != 0x41: raise ValueError("unexpected offset opcode")
                p += 1
                offset, p = read_varuint(payload, p)
                if payload[p] != 0x0b: raise ValueError("unexpected end opcode")
                p += 1
            elif flag == 2:
                _memidx, p = read_varuint(payload, p)
                if payload[p] != 0x41: raise ValueError("unexpected offset opcode")
                p += 1
                offset, p = read_varuint(payload, p)
                if payload[p] != 0x0b: raise ValueError("unexpected end opcode")
                p += 1
            else:
                raise ValueError(f"unsupported data segment flag: {flag}")

            data_size, p = read_varuint(payload, p)
            data = payload[p : p + data_size]
            p += data_size
            segments.append((offset, data))

    # Ενοποίηση των data segments στη μνήμη
    max_end = max(o + len(d) for o, d in segments)
    mem = bytearray(max_end + 1024)
    for o, d in segments:
        mem[o : o + len(d)] = d

    # Διαβάζουμε το πρόγραμμά μας
    prog = []
    for idx in range(count):
        off = base + idx * 4
        prog.append(struct.unpack_from("<I", mem, off)[0])
    return prog

def run_vm(prog: list[int], inp: bytes) -> tuple[bool, int]:
    tape = bytearray(256)
    ptr = 0
    ip = 0
    in_idx = 0
    
    while True:
        op = prog[ip]
        if op == OP_HALT_OK:
            return True, in_idx
        if op == OP_HALT_FAIL:
            return False, in_idx
        
        if op == OP_SET_CELL_IMM:
            tape[ptr] = prog[ip + 1] & 0xff
            ip += 2
            continue
        if op == OP_SET_PTR_IMM:
            ptr = prog[ip + 1] & 0xff
            ip += 2
            continue
        if op == OP_READ_INPUT:
            tape[ptr] = inp[in_idx]
            in_idx += 1
            ip += 1
            continue
        if op == OP_COPY_CELL:
            src = prog[ip + 1] & 0xff
            tape[ptr] = tape[src]
            ip += 2
            continue
        if op == OP_INC_CELL:
            tape[ptr] = (tape[ptr] + 1) & 0xff
            ip += 1
            continue
        if op == OP_DEC_CELL:
            tape[ptr] = (tape[ptr] - 1) & 0xff
            ip += 1
            continue
        if op == OP_JZ:
            hi = prog[ip + 1] & 0xff
            lo = prog[ip + 2] & 0xff
            target = (hi << 8) | lo
            if tape[ptr] == 0:
                ip = target
            else:
                ip += 3
            continue
            
        raise RuntimeError(f"unknown op {op} at {ip}")

def solve(wasm_path: Path, length: int = 20) -> bytes:
    prog = load_program(wasm_path)
    sol = bytearray([0] * length)

    for i in range(length):
        found = None
        for b in range(256):
            sol[i] = b
            ok, reads = run_vm(prog, bytes(sol))
            if i < length - 1:
                # Αν η μηχανή διάβασε και άλλο χαρακτήρα, περάσαμε το current check!
                if reads > i + 1:
                    found = b
                    break
            else:
                # Στον τελευταίο χαρακτήρα αρκεί το ok flag
                if ok:
                    found = b
                    break

        if found is None:
            raise RuntimeError(f"no byte found at position {i}")

    return bytes(sol)

def main() -> None:
    wasm_path = Path("5245e7a6")
    flag = solve(wasm_path)
    print(flag.decode("latin1"))

if __name__ == "__main__":
    main()
```

### Συμπέρασμα
Μια φανταστική άσκηση που συνδυάζει WebAssembly, Custom Virtual Machines, και λίγη φαντασία στην επίλυση, καθώς η εφαρμογή του brute-forcing πατώντας πάνω στο behavior ("oracle") του VM γλίτωσε εκατοντάδες ώρες αναλύσεων reverse engineering κώδικα με το χέρι!

**ΤΟ FLAG ΕΙΝΑΙ:**
`vm5_all_the_w4y_down`
