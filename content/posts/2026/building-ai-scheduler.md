---
title: "Building an AI Scheduler for Telescope Time Allocation"
date: 2026-05-12T18:00:00+00:00
draft: false
author: "Antonio V. Franco"
description: "How I built an AI scheduler to replace our observatory's spreadsheet-based telescope time allocation system and prevent costly scheduling conflicts."
tags:
  - "AI Engineering"
  - "Scheduling"
  - "Optimization"
  - "Astronomy"
  - "Python"
categories:
  - "Engineering"
  - "AI"
showToc: true
ShowReadingTime: true
keywords:
  # Google SEO Keywords (30)
  - "telescope scheduling software"
  - "observatory management system"
  - "astronomy scheduling Python"
  - "telescope time allocation"
  - "observatory scheduling tool"
  - "astronomical observation planner"
  - "telescope booking system"
  - "research proposal scheduler"
  - "observatory resource management"
  - "astronomy Python tools"
  - "telescope optimization algorithm"
  - "celestial observation scheduling"
  - "astronomical facility management"
  - "telescope queue management"
  - "observatory automation software"
  - "scheduling algorithm astronomy"
  - "telescope observation planner"
  - "astronomical research tools"
  - "observatory scheduling conflicts"
  - "telescope time management"
  - "Python astronomy scheduler"
  - "observatory constraint satisfaction"
  - "telescope scheduling conflicts"
  - "astronomical observation optimization"
  - "research telescope scheduling"
  - "observatory workflow automation"
  - "telescope allocation system"
  - "astronomical scheduling Python"
  - "observatory scheduling best practices"
  - "telescope management software"
  # GEO/LLM SEO Keywords (30)
  - "how to build telescope scheduler"
  - "best Python library for observatory scheduling"
  - "telescope time allocation system example"
  - "how to schedule astronomical observations"
  - "observatory management software comparison"
  - "what is telescope scheduling software"
  - "how to prevent telescope scheduling conflicts"
  - "best practices for observatory time allocation"
  - "how to implement greedy scheduling algorithm"
  - "telescope scheduler with constraint validation"
  - "how to calculate celestial visibility windows"
  - "observatory scheduling with Pydantic validation"
  - "how to build astronomy scheduling API"
  - "best telescope allocation system for research"
  - "how to optimize telescope observation time"
  - "observatory scheduler with moon constraints"
  - "how to create Gantt chart for telescope schedule"
  - "telescope scheduling checkpoint system"
  - "how to handle priority-based telescope scheduling"
  - "observatory scheduling state machine design"
  - "how to validate astronomy proposal constraints"
  - "telescope scheduler architecture diagram"
  - "how to integrate weather data in observatory scheduler"
  - "best open source telescope scheduling software"
  - "how to build visibility calculator for astronomy"
  - "observatory scheduler with instrument constraints"
  - "how to reschedule telescope observations automatically"
  - "telescope time allocation greedy algorithm"
  - "how to deploy astronomy scheduler on GitHub Pages"
  - "observatory management system with AI optimization"
---

Last month I realised something embarrassing. Our observatory was still scheduling telescope time using a spreadsheet. Three different research teams had been assigned the same observation slots, and nobody noticed until the night of the observations. We lost an entire week of clear skies to coordination failures that could have been avoided with even a moderately competent scheduling system.

The spreadsheet approach had worked well enough when we were a small team handling a handful of proposals each month. But our observatory has grown. We now receive dozens of research proposals from multiple institutions, each with specific target coordinates, exposure time requirements, and instrument preferences. The old system simply could not keep up. Excel was not going to cut it any longer.

I needed something that could handle conflicting constraints, respect research priorities, and (most importantly) never double-book the telescope. What I built over the following weeks became a modest but effective AI scheduler that has already saved us from three confirmed scheduling conflicts. Here is what I built, why I made certain design decisions, and what I learned along the way.

## The Problem

Scheduling telescope time is fundamentally a constraint satisfaction problem with competing priorities. You have multiple research proposals, each with different celestial targets and exposure time requirements. You have priority rankings that determine which proposals get preference when conflicts arise. You have visibility windows (celestial objects are not always observable from your location, and their visibility changes throughout the night and across seasons). You have instrument constraints (some proposals need the optical camera, others need the spectrograph, and you cannot use both simultaneously on the same telescope). And you have moon phases and weather conditions that affect observation quality.

The naive approach is first-come-first-served. A researcher submits a proposal, you find available time slots that match their visibility windows, and you book them. This approach fails spectacularly when a priority‑9 proposal (for example, a time‑critical supernova follow‑up that must happen within forty‑eight hours) arrives after a priority‑3 proposal (a general survey project with flexible timing) has already booked all the good slots. The high‑priority observation gets squeezed into suboptimal conditions, or worse, cannot be scheduled at all.

I considered several approaches before settling on a greedy algorithm that processes high‑priority proposals first. The algorithm looks at all proposals, sorts them by priority (highest first), and then schedules each proposal into the best available time slots that satisfy its visibility and instrument requirements. If a lower‑priority proposal has already claimed a slot that a higher‑priority proposal needs, the algorithm reclaims that slot and reschedules the lower‑priority proposal elsewhere.

This greedy approach is not globally optimal. You might miss a better overall arrangement by committing to early scheduling decisions. A higher‑priority proposal might consume a block of time that, if divided differently, could have accommodated two slightly lower‑priority proposals with better overall science output. But the greedy algorithm is fast, understandable, and works well enough for most real‑world cases at our observatory. The whole system runs in under ten seconds for a typical week's schedule. That speed matters when you need to regenerate a schedule because someone just submitted a priority‑10 target‑of‑opportunity request.

## System Architecture

The scheduler has four main components organised in layers. Each layer has a clear responsibility, and the boundaries between layers are strictly enforced. This separation of concerns matters enormously when things break. And things do break. Rate limits get hit. APIs change their response formats without warning. Someone submits a proposal with coordinates in the wrong format (degrees instead of hours, or decimal degrees instead of degrees/minutes/seconds). When a failure occurs, a clean architecture helps you localise the problem quickly.

The user interface layer sits at the top. It provides three access methods: a command‑line interface for quick scheduling runs, an API for integration with our existing proposal management system, and a web dashboard for visual inspection of schedules. The dashboard is the newest addition and has already become the most popular way to interact with the scheduler.

Beneath the user interface layer lies the agent layer. This is where the core logic lives. The proposal agent parses incoming proposals, validates their fields, and prepares them for scheduling. The visibility agent calculates when each target is observable from our observatory's latitude and longitude. The weather agent (still under development) integrates forecast data to avoid scheduling observations during predicted cloud cover. The optimisation agent runs the greedy scheduling algorithm and handles conflict resolution.

Below the agent layer is the tool layer, which provides external integrations. The LLM client handles optional natural‑language processing (for example, extracting proposal details from free‑text emails). The AstroQuery module interfaces with astronomical databases to resolve target names to coordinates. The weather API client fetches forecast data from national meteorological services.

At the bottom sits the data layer, which handles persistence and validation. Pydantic models define the shape of every data structure and validate incoming data before it reaches the business logic. Checkpoint managers save intermediate scheduling states so that long optimisation runs can resume after a crash. Log files record every scheduling decision for later audit and debugging.

![System architecture diagram showing the four‑layer design with clear separation of concerns. The user interface layer sits on top, followed by the agent layer, then the tool layer, and finally the data layer at the base.](/images/ai-scheduler/architecture_diagram.png)

*Figure 1: System architecture diagram showing the four‑layer design with clear separation of concerns.*

This layered architecture has saved me countless hours of debugging. When a schedule looks wrong, I can inspect the state at each layer and see exactly where the problem originated. Was the visibility calculation incorrect? That is a tool layer or agent layer issue. Did the optimisation produce overlapping slots? That is an agent layer issue. Did a proposal fail validation? That is a data layer issue. The separation makes systematic debugging possible.

## The Models

Everything starts with Pydantic models. These models define the shape of every piece of data that moves through the system. They validate input at the boundaries, provide clear error messages when something is wrong, and serve as living documentation of what the system expects.

Here is the Proposal model that represents a research proposal:

```python
from pydantic import BaseModel, Field, field_validator
from typing import List, Optional
from datetime import datetime

class Coordinates(BaseModel):
    ra: float = Field(..., ge=0, le=360, description="Right Ascension in degrees")
    dec: float = Field(..., ge=-90, le=90, description="Declination in degrees")
    name: Optional[str] = None

class Proposal(BaseModel):
    id: str
    researcher: str
    email: str
    title: str
    targets: List[Coordinates]
    exposure_hours: float = Field(..., gt=0)
    priority: int = Field(..., ge=1, le=10)
    deadline: Optional[datetime] = None
    instrument: str = "optical_camera"
    is_time_critical: bool = False
    moon_constraints: str = "any"  # "dark", "any", "moon_up"
```

The field validators catch problems early. Someone submits a proposal with a right ascension of four hundred degrees? They receive a clear error message explaining that the value must be between zero and three hundred sixty degrees. Someone forgets to specify an exposure time? The model rejects the proposal with a message indicating that exposure_hours must be greater than zero. Without these validators, the same error might propagate three layers down before causing a cryptic crash in the visibility calculator.

I also use custom validators for more complex checks. For example, a validator ensures that proposals marked as time‑critical have a deadline set, because a time‑critical observation without a deadline makes no sense. Another validator checks that the list of targets is not empty. A third validator normalises instrument names (accepting both "spectrograph" and "spectrometer" and mapping them to a canonical value).

The models have evolved over time. Originally, the Proposal model had no moon_constraints field. That omission caused a problem when a researcher submitted a proposal requiring dark skies (no moon) and the scheduler assigned it to a night with a bright full moon. Adding the field required updating the validation logic, the constraint checker, and the scheduling algorithm. But because the model defined the shape of the data in one place, the changes were localised and manageable.

## Calculating Visibility

You cannot schedule a telescope to look at something that is below the horizon. Worse, you cannot schedule a telescope to look at something that is behind the moon. The visibility calculator determines when celestial objects are observable from your specific geographic location. This calculation is more complex than it might seem at first glance.

I use the Astropy library for the heavy lifting. Astropy handles coordinate transformations (converting between equatorial coordinates and altitude‑azimuth coordinates), atmospheric refraction corrections, and lunar ephemeris calculations. These are calculations that I absolutely do not want to implement myself. Celestial mechanics is a field full of subtle corrections and edge cases, and getting it wrong by even a small fraction of a degree can render a schedule useless.

The visibility calculator takes a target's coordinates (right ascension and declination) and a candidate time, then returns whether the target is observable at that time and a quality score representing how good the observing conditions are likely to be.

```python
from astropy.coordinates import EarthLocation, SkyCoord, AltAz
from astropy.time import Time
import astropy.units as u

class VisibilityCalculator:
    def __init__(self, latitude: float, longitude: float, elevation: float):
        self.location = EarthLocation(lat=latitude*u.deg, 
                                       lon=longitude*u.deg, 
                                       height=elevation*u.m)
    
    def is_observable(self, ra: float, dec: float, time: datetime) -> tuple[bool, float]:
        """Check if object is observable at given time.
        
        Returns (is_observable, quality_score) where quality_score is 0-100.
        """
        coord = SkyCoord(ra=ra*u.deg, dec=dec*u.deg)
        altaz_frame = AltAz(obstime=Time(time), location=self.location)
        altaz = coord.transform_to(altaz_frame)
        
        altitude = altaz.alt.deg
        
        # Below horizon
        if altitude < 0:
            return False, 0
        
        # Poor airmass (too close to horizon)
        if altitude < 30:
            return False, 0
        
        # Quality based on altitude (higher = better)
        quality = min(100, (altitude - 30) * 2)
        return True, quality
```

The calculator enforces two simple rules. First, the target must be above the horizon (altitude greater than zero degrees). Second, the target must be at an altitude of at least thirty degrees to avoid the worst atmospheric effects. Light from a low‑altitude target passes through much more atmosphere than light from a high‑altitude target, resulting in blurred images and reduced signal‑to‑noise ratios.

The quality score scales linearly with altitude from thirty degrees (quality zero) to eighty degrees (quality one hundred). A target at eighty degrees altitude is nearly overhead and provides the best possible observing conditions. A target at thirty‑five degrees receives a quality score of ten, which is barely acceptable. This scoring system guides the scheduling algorithm toward better time slots when multiple options are available.

![Visibility windows for Messier 15 over a seven‑day period. The chart shows observable hours per night and quality scores. Night one shows 4.7 observable hours with quality seventy‑four. Night two shows 3.8 hours with quality fifty‑one. Night three shows 5.0 hours with quality seventy‑two.](/images/ai-scheduler/visibility_windows.png)

*Figure 2: Visibility windows for Messier 15 over a seven‑day period.*

I learned an important lesson while building the visibility calculator. The naive approach of checking visibility once per hour is insufficient. A target might be observable for only twenty minutes between moonrise and astronomical twilight. The scheduler now checks visibility in fifteen‑minute increments and aggregates contiguous observable blocks into windows. This finer granularity captures short but valuable observing opportunities that would otherwise be missed.

## The Optimisation Workflow

The scheduler uses a state machine to manage the optimisation process. Each state has a clear responsibility, and transitions between states happen based on validation results. This state machine design makes the optimisation logic transparent and testable.

The workflow begins in the Start state. From there, it transitions to Initialize, where the system loads all proposals, precomputes visibility windows for every target, and sorts the proposals by priority. Initialisation also loads any existing schedule (if this is a rescheduling run) and applies any manual overrides from observatory staff.

After initialisation, the workflow moves to Generate Candidates. In this state, the scheduler creates candidate time slots for each proposal based on its visibility windows and the availability of required instruments. A proposal with multiple targets generates multiple candidate slots, one for each target. A proposal with six hours of exposure time might be split into multiple shorter slots across different nights if no single night provides six continuous observable hours.

The workflow then enters Validate Constraints. This state checks the current set of candidate slots against three categories of constraints. Hard constraints must be satisfied for a schedule to be valid at all. Soft constraints can be violated but with a penalty. The validator produces a list of constraint violations, which can be either "valid" (no violations) or "invalid" (some violations exist).

If violations exist, the workflow moves to Score Solutions. This state evaluates the current schedule according to an objective function that balances multiple goals. Completing high‑priority proposals is weighted heavily. Maximising total science output (exposure hours scheduled) is weighted moderately. Avoiding schedule fragmentation (short, inefficient slots) is weighted lightly. The scoring function produces a numerical score that allows the scheduler to compare different candidate schedules.

From Score Solutions, the workflow proceeds to Check Completion. This state decides whether the optimisation should continue or terminate. Termination occurs when the schedule has no hard constraint violations and the score has not improved over several iterations. If the scheduler decides to continue, it moves to Refine; if it decides it is done, it moves to Finalize.

The Refine state attempts to fix constraint violations by shifting or removing conflicting slots. The refinement logic is simple but effective. For overlapping slots on the same instrument, the refiner examines both proposals and reschedules the lower‑priority one to a different time, if possible. If no alternative time exists, the refiner removes the lower‑priority slot entirely and logs a warning. For moon constraint violations, the refiner shifts the observation to a different night with appropriate moon conditions. After refinement, the workflow returns to Validate Constraints for another iteration.

When the workflow reached Finalize, the schedule is complete. This state builds the final schedule object, attaches metadata (timestamp, scheduler version, runtime statistics), and passes the schedule to the checkpoint manager for persistence. The workflow then moves to the End state and terminates.

![State machine diagram showing the optimisation workflow with conditional transitions. Start leads to Initialize, then to Generate Candidates, then to Validate Constraints, which branches to Score Solutions or Check Completion, eventually leading to Refine or Finalize, and finally End.](/images/ai-scheduler/workflow_diagram.png)

*Figure 3: State machine diagram showing the optimisation workflow.*

I chose this state machine structure because it makes debugging tractable. When a schedule looks wrong, I can inspect the state at each step and see where things went off the rails. Was the validation state correct? Did the refinement step introduce new violations? The answers to these questions point directly to the problematic component.

## Handling Constraints

The constraint validator checks three main categories of constraints. Each category has its own validation function, and the validator aggregates violations from all categories before returning a result.

The first category is no overlapping slots on the same instrument. This is the most obvious constraint: a telescope cannot observe two different targets at the same time. The validation function iterates through all scheduled slots, groups them by instrument, and checks for temporal overlaps within each group.

```python
def validate_no_overlap(slots: List[ScheduledSlot]) -> List[ConstraintViolation]:
    violations = []
    for i, slot1 in enumerate(slots):
        for slot2 in slots[i+1:]:
            if slot1.instrument != slot2.instrument:
                continue
            # Check time overlap
            if slot1.start_time < slot2.end_time and slot2.start_time < slot1.end_time:
                violations.append(ConstraintViolation(
                    violation_type="overlap",
                    slot_ids=[slot1.proposal_id, slot2.proposal_id],
                    message=f"{slot1.proposal_id} and {slot2.proposal_id} overlap on {slot1.instrument}"
                ))
    return violations
```

This function is deliberately simple. It compares every pair of slots on the same instrument and checks whether their time intervals intersect. A more efficient algorithm would use interval trees, but the number of slots per schedule (typically fewer than fifty) makes the quadratic approach perfectly acceptable.

The second category is instrument compatibility. Some proposals require specific instruments that may not be available on all nights. The spectrograph might be undergoing maintenance on Tuesdays. The optical camera might be scheduled for a different user during certain hours. The validator checks each proposal's instrument request against the instrument availability calendar and flags any mismatches.

The third category is moon constraints. Some observations require dark time (when the moon is below the horizon or in a thin crescent phase) because moonlight contaminates the measurements. Other observations can run with the moon up (for example, bright targets or spectroscopy where the lunar background can be subtracted). The validator checks each slot's scheduled time against the moon phase and altitude, then verifies that the proposal's moon_constraints setting is satisfied.

When violations exist, the refiner attempts to fix them. The refiner's strategy is to shift slots to non‑conflicting times first, then to remove lower‑priority observations if shifting is impossible. This approach is not elegant, but it converges quickly. In practice, after three to five refinement iterations, the schedule typically satisfies all hard constraints.

I added logging to the refiner after an incident where the system entered an infinite refinement loop. Two proposals had mutually exclusive visibility windows (proposal A was visible only when proposal B was not, and vice versa), and the refiner kept swapping them without progress. The logging revealed the problem within minutes. I then added a cycle detection mechanism that terminates refinement after a maximum number of iterations.

## Checkpointing Long Runs

Optimisation can take time, especially when scheduling a full month of observations with dozens of proposals. I did not want to lose progress if the process crashed or was killed by an administrator. The checkpoint manager saves the scheduler's state after each scoring iteration, allowing a crashed run to resume from the last checkpoint rather than starting from scratch.

```python
from pathlib import Path
import json
from datetime import datetime

class CheckpointManager:
    def __init__(self, checkpoint_dir: str, keep: int = 5):
        self.checkpoint_dir = Path(checkpoint_dir)
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)
        self.keep = keep
    
    def save(self, state: dict, checkpoint_id: str):
        """Save checkpoint with atomic write."""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"checkpoint_{checkpoint_id}_{timestamp}.json"
        path = self.checkpoint_dir / filename
        
        # Atomic write via temp file + rename
        temp_path = path.with_suffix(".tmp")
        with open(temp_path, "w") as f:
            json.dump(state, f, indent=2, default=str)
        temp_path.rename(path)
        
        # Cleanup old checkpoints
        self._cleanup(checkpoint_id)
```

The atomic write pattern (write to a temporary file, then rename it to the final name) prevents corruption if the process dies in the middle of writing. Without this pattern, a crash during the write operation could leave the checkpoint file half‑written and unusable. I learned this lesson the hard way on a previous project where corrupted checkpoint files caused more trouble than the crashes they were meant to mitigate.

The checkpoint manager also implements automatic cleanup, keeping only the most recent five checkpoints for each scheduling run. Older checkpoints are deleted to save disk space. The keep parameter is configurable; for long‑running optimisation jobs (for example, scheduling an entire semester), I increase it to twenty to provide more recovery points.

Each checkpoint contains the complete state of the scheduler. This includes the list of proposals, the current schedule, the scoring history, the iteration count, and the random number generator state (for reproducible results). Restoring from a checkpoint is a matter of loading the JSON file and reinitialising the scheduler with the saved state.

I initially resisted adding checkpointing, thinking that ten seconds was too short to justify the complexity. But then I added a new feature (weather simulation) that increased runtime to several minutes, and I was grateful for the existing checkpoint infrastructure. The lesson is that checkpointing is worth the extra code even when runs are short, because runs have a way of becoming longer over time.

## Visualising the Schedule

A list of time slots with start times, end times, and proposal IDs is not easy to parse. I built two visualisation tools to make schedules comprehensible at a glance.

The first visualisation is a text‑based Gantt chart that renders in the terminal. This tool is useful for quick inspections during development and for researchers who prefer working on the command line. The Gantt chart shows each instrument on a separate row, with time progressing from left to right. Each proposal appears as a coloured block (represented by different characters in the text version) spanning its scheduled duration.

The second visualisation is a web‑based interactive chart built with a JavaScript plotting library. This chart allows users to zoom in on specific time ranges, click on blocks to see proposal details, and toggle instruments on and off. The web dashboard has become the primary interface for most observatory staff.

Both visualisations helped me spot bugs that unit tests missed. In one case, the Gantt chart showed overlapping slots that the validator had somehow missed (a bug in the validator's time comparison logic). In another case, the chart revealed that all high‑priority proposals were being scheduled in the first two nights of the week, leaving the remaining five nights empty. This imbalance was not technically a violation, but it was clearly suboptimal, and it led me to improve the scoring function to encourage better temporal distribution.

The visualisation also includes statistics: total hours scheduled, telescope utilisation percentage, average slot duration, and priority distribution. These statistics provide a quick sanity check. If utilisation is below fifty percent on a clear week, something is probably wrong. If average slot duration is thirty minutes (the minimum configurable length) for all slots, the scheduler might be fragmenting observations unnecessarily.

![Gantt chart visualisation showing telescope time allocation across instruments and proposals. The optical camera row shows several blocks representing observations of Messier 15 and Messier 31. The spectrograph row shows blocks representing observations of the Orion Nebula.](/images/ai-scheduler/gantt_chart.png)

*Figure 4: Gantt chart visualisation showing telescope time allocation.*

![Priority‑based scheduling chart showing hours allocated per proposal and time distribution. Proposal DEMO‑001 (priority 8) receives six hours, Proposal DEMO‑002 (priority 6) receives three hours, Proposal DEMO‑003 (priority 9) receives three hours.](/images/ai-scheduler/priority_chart.png)

*Figure 5: Priority‑based scheduling chart showing hours allocated per proposal.*

## What the Output Looks Like

Running the scheduler on three sample proposals produces a complete schedule package. The package includes the observation timeline, instrument assignments, and supporting metadata.

For the three sample proposals (a priority‑9 supernova follow‑up, a priority‑8 variable star survey, and a priority‑6 galaxy imaging project), the scheduler produces a schedule with two to three observation slots. The priority‑9 proposal receives the best available time slots (high altitude, dark skies, minimal moon interference). The priority‑8 proposal receives the next best slots. The priority‑6 proposal receives whatever remains.

Telescope utilisation typically ranges from seventy to eighty percent. The remaining twenty to thirty percent accounts for telescope maintenance, calibration observations, and weather buffers. This utilisation rate is higher than what we achieved with manual spreadsheet scheduling (which averaged around fifty‑five percent) because the scheduler identifies and fills short gaps that human schedulers tended to overlook.

The scheduler guarantees no overlapping slots on the same instrument. This guarantee is enforced by the constraint validator and refinement loop, and I have never seen a violation in production. The scheduler also guarantees that visibility windows are respected for all targets. Every observation is scheduled only during times when its target is above thirty degrees altitude.

The output exports to multiple formats. JSON format is used for integration with the web dashboard and for archiving. CSV format is used for import into legacy tools that still expect spreadsheets. A human‑readable text format is used for email notifications to researchers. Each format includes the same core information (start time, end time, instrument, target coordinates, proposal ID, researcher name) but presents it differently.

I also added an experimental natural‑language output that writes the schedule as a paragraph of English text. "On Monday night, from 22:00 to 02:00, the optical camera will observe Messier 15 for Dr. Smith's variable star survey. The spectrograph will then observe the Orion Nebula from 02:30 to 05:30 for Dr. Garcia's spectroscopy proposal." This format is surprisingly useful for busy researchers who just want a quick summary without parsing tables.

## Trade‑offs

This scheduler is not the optimal solution to the scheduling problem. A constraint programming solver (such as Z3 or Google OR‑Tools) would find better arrangements. These solvers use sophisticated algorithms (branch and bound, constraint propagation, and sometimes integer linear programming) to explore the solution space more thoroughly than my greedy algorithm ever could.

But I chose not to use a constraint solver for three reasons. First, solvers are slower. A constraint solver might take minutes to find an optimal schedule for a week's worth of proposals, whereas my greedy algorithm returns a good schedule in seconds. When a last‑minute target‑of‑opportunity proposal arrives, seconds matter more than optimality.

Second, solvers are harder to debug. When a constraint solver produces a surprising result, understanding why it made that decision can be difficult. The solver's internal state is complex, and the mapping from constraints to decisions is opaque. With my greedy algorithm, I can inspect each decision in order and see exactly why a particular slot was chosen.

Third, constraint solvers are overkill for our scale. Our observatory handles dozens of proposals, not thousands. The solution space is large enough that a naive exhaustive search would be impossible, but small enough that a well‑tuned greedy algorithm performs admirably. I chose "works well enough and I understand it" over "mathematically optimal but opaque". That is a judgment call, and your mileage may vary.

There are other trade‑offs as well. The scheduler assumes that all proposals are independent, which is not always true. Some proposals are complementary (observing the same target with different instruments) and should be scheduled together. Others are mutually exclusive (different research groups requesting the same instrument on the same night) and require arbitration. The current system handles the latter but not the former.

The scheduler also assumes that all observations are equally valuable per hour, which is false. The first hour of observation on a new target is often more valuable than the tenth hour, because the marginal scientific return decreases. A more sophisticated scheduler would account for diminishing returns and might prioritise many short observations over a few long ones. I have not implemented this because the required utility functions would be highly subjective.

## What Is Next

Weather integration is at the top of my list. The current system assumes clear skies, which is optimistic for most observatories. I am adding a weather agent that fetches cloud cover forecasts, seeing conditions (atmospheric turbulence), and humidity predictions. The scheduler will then avoid scheduling high‑priority observations during predicted poor conditions.

The weather agent presents an interesting challenge. Forecasts are probabilistic, not deterministic. A schedule that assumes clear skies might be useless if the forecast changes. I am experimenting with a two‑stage approach: first, generate a nominal schedule using the current forecast; second, regenerate the schedule in real time as the forecast updates. The checkpointing system will make this regeneration efficient.

I also want to add a web dashboard that shows the schedule in an interactive calendar view. Researchers should be able to see their scheduled observations without SSHing into the server or downloading a CSV file. The dashboard should also allow researchers to request changes (for example, swapping time slots with another group) and automatically check whether the requested change violates any constraints.

The dashboard is partially built but needs more work. The current version shows a static Gantt chart but does not support interactivity. I plan to add drag‑and‑drop rescheduling, email notifications for schedule changes, and a proposal submission form that validates inputs before they reach the scheduler.

Another improvement I am considering is a learning component that tracks which schedules were actually executed versus which were planned. If the scheduler consistently overestimates the time needed for certain types of observations, it should adjust its exposure time estimates accordingly. This would require collecting execution logs from the telescope control system and feeding them back into the scheduler's models.

The code is on GitHub under an open‑source licence. Other observatories have already started using it, and I have received several pull requests adding support for different telescope control systems and instrument configurations. If you have built something similar (or if you think I missed something obvious), I would love to hear about it.

---

## Lessons Learned

Start with the simplest thing that could possibly work. The first version of the greedy scheduler was fifty lines of code. It had no checkpointing, no visualisation, and no constraint validation beyond basic overlap detection. It was barely adequate, but it worked well enough to replace the spreadsheet. From that foundation, I added features incrementally based on real needs rather than hypothetical ones.

Pydantic models pay for themselves in avoided bugs and clearer error messages. Every time a proposal fails validation, someone receives a helpful error message explaining what is wrong. Before Pydantic, the same error would cause a cryptic crash deep in the visibility calculator, and the researcher would have to email me for help. The models have reduced my support email volume by about eighty percent.

Checkpointing is worth the extra code when runs take more than a few seconds. I added checkpointing when runs were still short, anticipating that they would become longer. That anticipation paid off when weather simulation increased runtime to several minutes. Without checkpointing, every crash would lose minutes of work. With checkpointing, crashes lose at most a few seconds.

Visualisation is not just for users; it is a debugging tool. The Gantt chart revealed scheduling anomalies that no unit test would have caught. When you can see the schedule, problems become obvious in ways that raw data never can. I now add visualisation to every data processing project, even if the intended audience is just me.

Build something. Ship it. Learn from it. The scheduler is not perfect, but it is better than what we had, and it has already saved our observatory from several scheduling disasters. The next version will be better, and the version after that will be better still. That is the only trajectory that matters.

---

*I am Antonio V. Franco, an AI Engineer focusing on RAG, LLMs and SLMs, agents, and automations. If you would like to get in touch with me, I will be happy to talk via email: contact@antoniovfranco.com*
