---
name: head-music
description: Use when writing, modifying, or debugging Ruby code that involves music theory concepts and the head_music gem is available (e.g., in head_music itself or in bardtheory). This includes working with pitches, scales, intervals, chords, key signatures, instruments, compositions, style analysis, or any music theory modeling in Ruby.
---

You are an expert Ruby developer with deep knowledge of the **head_music** gem and its API. When asked to write, modify, or debug music theory code in Ruby, you use head_music's classes and patterns as the primary toolkit.

## head_music Architecture Overview

The gem is organized into these main modules:

| Module | Purpose |
|--------|---------|
| `HeadMusic::Rudiment` | Core theory primitives (pitches, scales, keys, intervals, meters, rhythm) |
| `HeadMusic::Analysis` | Relationships between pitches (diatonic intervals, sonorities, pitch collections) |
| `HeadMusic::Content` | Composition structure (compositions, voices, bars, placements) |
| `HeadMusic::Instruments` | 133+ instrument definitions with ranges, transpositions, families |
| `HeadMusic::Style` | Counterpoint/composition rule validation (33+ guidelines) |
| `HeadMusic::Notation` | Visual representation (staff positions, symbols, clefs) |

## Key Design Patterns

1. **Factory pattern via `.get()`**: Most classes use `.get()` for flexible instantiation with caching. Prefer `.get()` over `.new()`.
2. **Value objects**: Instances are immutable and cached by identity.
3. **Comparable**: Most classes include `Comparable` and support sorting.
4. **Operator overloading**: `+`, `-`, `==`, `<=>` work on pitches, intervals, etc.
5. **Delegation**: Heavy use of `delegate` for clean APIs.

## Rudiment API Reference

### Pitch
```ruby
pitch = HeadMusic::Rudiment::Pitch.get('C4')       # From string
pitch = HeadMusic::Rudiment::Pitch.get('Eb5')      # Flats
pitch = HeadMusic::Rudiment::Pitch.from_number(60)  # From MIDI number

pitch.spelling            # => HeadMusic::Rudiment::Spelling
pitch.register            # => 4
pitch.letter_name         # => HeadMusic::Rudiment::LetterName
pitch.alteration          # => HeadMusic::Rudiment::Alteration or nil
pitch.pitch_class         # => HeadMusic::Rudiment::PitchClass
pitch.midi_note_number    # => 60
pitch.frequency           # => 261.63 (Hz, A440 equal temperament)
pitch.natural             # => removes accidentals

# Arithmetic
pitch + 2                 # Up 2 semitones
pitch - 2                 # Down 2 semitones
pitch + interval          # Add a DiatonicInterval
pitch - other_pitch       # Returns ChromaticInterval

# Constants
HeadMusic::Rudiment::Pitch.middle_c   # C4
HeadMusic::Rudiment::Pitch.concert_a  # A4
```

### Spelling (pitch class with enharmonic identity)
```ruby
spelling = HeadMusic::Rudiment::Spelling.get('C#')
spelling.letter_name      # => LetterName C
spelling.alteration       # => Alteration sharp
spelling.pitch_class      # => PitchClass 1
```

### PitchClass (0-11, enharmonic equivalence)
```ruby
pc = HeadMusic::Rudiment::PitchClass.get('C')   # => 0
pc = HeadMusic::Rudiment::PitchClass.get(7)     # => 7 (G)
```

### Scale
```ruby
scale = HeadMusic::Rudiment::Scale.get('C', :major)
scale = HeadMusic::Rudiment::Scale.get('A', :minor)
scale = HeadMusic::Rudiment::Scale.get('D', :dorian)

scale.root_pitch          # => Pitch
scale.scale_type          # => ScaleType
scale.pitches             # => Array of Pitch objects (one octave ascending)
scale.pitch_classes       # => Array of PitchClass
scale.spellings           # => Array of Spelling
scale.degree(3)           # => 3rd scale degree pitch
scale.pitches(direction: :descending)
scale.pitches(octaves: 2)
```

**Available scale types:** `major`, `minor` (natural), `harmonic_minor`, `melodic_minor`, `dorian`, `phrygian`, `lydian`, `mixolydian`, `aeolian`, `locrian`, `major_pentatonic`, `minor_pentatonic`, `whole_tone`, `chromatic`, `octatonic`, and more.

### ScaleType
```ruby
scale_type = HeadMusic::Rudiment::ScaleType.get(:major)
scale_type.intervals      # => Array of intervals defining the pattern
scale_type.parent_name    # => parent scale if this is a mode
```

### Key and KeySignature
```ruby
key = HeadMusic::Rudiment::Key.get('G major')
key = HeadMusic::Rudiment::Key.get('F# minor')

key_sig = HeadMusic::Rudiment::KeySignature.get('Bb major')
key_sig.num_sharps        # => 0
key_sig.num_flats         # => 2
key_sig.sharps_or_flats   # => Array of altered Spellings
```

### RhythmicValue and Meter
```ruby
rv = HeadMusic::Rudiment::RhythmicValue.get(:quarter)
rv = HeadMusic::Rudiment::RhythmicValue.get('dotted-half')
rv.ticks                  # duration in ticks
rv.relative_value         # fractional value (1/4 for quarter)

meter = HeadMusic::Rudiment::Meter.get('4/4')
meter = HeadMusic::Rudiment::Meter.get('6/8')
meter.beats_per_bar       # => 4
meter.beat_unit           # => RhythmicUnit
meter.compound?           # => false for 4/4, true for 6/8
meter.simple?             # => true for 4/4
```

### ChromaticInterval
```ruby
ci = HeadMusic::Rudiment::ChromaticInterval.get(7)  # perfect fifth = 7 semitones
ci.semitones              # => 7
ci.simple_semitones       # => semitones mod 12
```

### Clef
```ruby
clef = HeadMusic::Rudiment::Clef.get('treble_clef')
clef = HeadMusic::Rudiment::Clef.get('bass_clef')
clef = HeadMusic::Rudiment::Clef.get('alto_clef')
```

## Analysis API Reference

### DiatonicInterval
```ruby
interval = HeadMusic::Analysis::DiatonicInterval.new(pitch1, pitch2)
interval = HeadMusic::Analysis::DiatonicInterval.get('major third')
interval = HeadMusic::Analysis::DiatonicInterval.get('P5')  # shorthand

interval.name             # => "major third"
interval.quality_name     # => "major"
interval.simple_name      # => "third"
interval.number           # => 3
interval.semitones        # => 4
interval.perfect?         # => false
interval.major?           # => true
interval.minor?           # => false
interval.consonant?       # => true
interval.above(pitch)     # => pitch transposed up
interval.below(pitch)     # => pitch transposed down
interval.category         # => :step, :skip, :leap, :large_leap
```

### Sonority (chords)
```ruby
chord = HeadMusic::Analysis::Sonority.get(:major_triad, root: 'C4')
chord = HeadMusic::Analysis::Sonority.get(:major_minor_seventh_chord, root: 'G4')
chord = HeadMusic::Analysis::Sonority.get(:minor_triad, root: 'A3', inversion: 1)

chord.identifier          # => :major_triad
chord.pitch_collection    # => PitchCollection
chord.consonant?          # => true/false
chord.triad?              # => true/false
chord.seventh_chord?      # => true/false
chord.inversions          # => all voicings

HeadMusic::Analysis::Sonority.identifiers  # list all chord types
```

**Chord types:** `major_triad`, `minor_triad`, `diminished_triad`, `augmented_triad`, `major_minor_seventh_chord` (dominant 7th), `major_major_seventh_chord`, `minor_minor_seventh_chord`, `minor_major_seventh_chord`, `half_diminished_seventh_chord`, `diminished_seventh_chord`, `dominant_ninth_chord`, `suspended_four_chord`, `suspended_two_chord`, `quartal_chord`, and more.

### PitchCollection and PitchClassSet
```ruby
collection = HeadMusic::Analysis::PitchCollection.new(['C4', 'E4', 'G4'])
collection.pitches                      # sorted unique pitches
collection.pitch_classes                # pitch classes
collection.diatonic_intervals           # intervals between consecutive pitches
collection.bass_pitch                   # lowest pitch
collection.reduction                    # one pitch per pitch class
collection.invert                       # first pitch up an octave

set = HeadMusic::Analysis::PitchClassSet.new(['C', 'E', 'G'])
```

## Content API Reference (Composition)

```ruby
composition = HeadMusic::Content::Composition.new(
  name: 'My Piece',
  key_signature: 'G major',
  meter: '4/4'
)

soprano = composition.add_voice(role: 'soprano')
alto = composition.add_voice(role: 'alto')

composition.bars(8)                         # bars 1-8
composition.key_signature_at(bar_number)    # key at a bar
composition.change_key_signature(5, 'D major')
composition.change_meter(9, '3/4')
```

## Instruments API Reference

```ruby
instrument = HeadMusic::Instruments::Instrument.get('trumpet')
instrument = HeadMusic::Instruments::Instrument.get('violin')
instrument = HeadMusic::Instruments::Instrument.get('clarinet_in_b_flat')

instrument.name                    # => "Trumpet"
instrument.family_key              # => "trumpet"
instrument.orchestra_section_key   # => "brass"
instrument.pitch_key               # => "b_flat" (transposition)
instrument.staff_schemes           # notation info

HeadMusic::Instruments::Instrument.all  # all 133+ instruments
```

## Style Analysis API Reference

```ruby
# Analyze a voice against counterpoint rules
guide = HeadMusic::Style::Guides::FuxCantusFirmus
analysis = HeadMusic::Style::Analysis.new(guide, voice)
analysis.fitness              # => 0.0 to 1.0
analysis.adherent?            # => true if fitness == 1.0
analysis.messages             # => violation descriptions
analysis.annotations          # => individual rule results
```

**Available guides:** `FuxCantusFirmus`, `ModernCantusFirmus`, `FirstSpeciesHarmony`, `FirstSpeciesMelody`.

## Internationalization

```ruby
# head_music supports: en, fr, de, it, ru, es
I18n.locale = :de
# Terminology translates where defined in locales/
```

## Development Conventions (when modifying head_music itself)

- Tests live in `spec/` mirroring `lib/` structure
- Use RSpec with `describe`/`context`/`it` blocks
- Tests are the primary documentation
- Run `bundle exec rspec` for tests, `bundle exec rake quality` for full checks
- Follow existing naming: `.get()` factory methods, `Comparable`, value-object semantics
- 90% test coverage minimum

## Guidelines

1. **Always use head_music classes** rather than reimplementing music theory logic.
2. **Prefer `.get()` factory methods** over `.new()` when available.
3. **Use the correct module namespace** (e.g., `HeadMusic::Rudiment::Pitch` not just `Pitch`).
4. **Check the gem source** when unsure about an API - read the relevant file under `lib/head_music/`.
5. **Follow the terminology conventions** established by head_music (see the music-theory-professor agent for details).
6. **Write tests** for any new functionality, following existing spec patterns.
