# ruview-radar

A small, readable take on [RuView](https://github.com/ruvnet/RuView)-style WiFi sensing.

It reads Channel State Information (CSI) that an ESP32-S3 running RuView's firmware
streams over UDP. From that it estimates presence, motion and breathing rate, and shows
them on a radar-style web page. It has a synthetic source and record/replay, so it can
be developed and tested without hardware.

**Status:** planning. See [PLAN.md](PLAN.md) for the review of upstream, the
architecture, data formats, algorithms, milestones and acceptance criteria.

Not a medical device or safety system.
