# Hackathon 2026 Track 1-Michael Lee 10-3
# To run, just run the code. Incomplete prototype, no UI, etc. Just run code via python IDE
# Program description: generate, validate, optimize, and export two school timetables.
"""SmartTimetable v9.1.1 - compact final (Track 1). Stdlib only. Runs directly."""
# Standard-library tools: copying solutions, CSV output, math, randomness, and command-line arguments.
import copy,csv,math,random,sys
# Alias defaultdict to dd; missing keys automatically receive a default value.
from collections import defaultdict as dd

# The timetable has five school days and seven defined periods per day.
DAYS,PER=5,7
# Start/end times for each period; these durations are not all equal.
TIMES=["08:30-09:30","09:40-10:20","10:30-11:10","11:20-12:05","13:15-14:10","14:20-15:00","15:10-15:50"]
# Generate (day, period) pairs; Friday has only periods 1–5, giving 33 slots per group.
SLOTS=[(d,p) for d in range(1,6) for p in range(1,8 if d<5 else 6)]      # 33 slots/block
# CAP is the daily teacher limit; BLOCK courses need consecutive periods; PRI controls room-assignment order.
CAP=5; BLOCK={"CreWriting","CAS"}; PRI={"pe":0,"sci":1,"std":2}
# Convert each HH:MM-HH:MM string into a duration measured in minutes.
DUR=[(lambda a,b:int(b[:2])*60+int(b[3:])-int(a[:2])*60-int(a[3:]))(*t.split("-")) for t in TIMES]
# Japanese may only use periods that last 40 minutes.
JP={p for p in range(1,8) if DUR[p-1]==40}                               # 40-min periods
# P55 contains the 55-minute period(s); PE is restricted to this set.
P55={p for p in range(1,8) if DUR[p-1]==55}                              # 55-min = P5, first afternoon
# Map restricted courses to allowed periods; unrestricted courses can use any period.
CP={"Japanese":JP,"PE":P55}; ALLP=set(range(1,8))                        # course -> allowed periods

# Main configuration: teacher unavailability, room types, course requirements, and group curricula.
DATA={"unavail":{"T-He":[[4,1]],"T-Lu":[[2,6]]},
 "rooms":[{"id":"ZX215","types":["sci"]},{"id":"ZX209","types":["sci"]},{"id":"ZX210","types":["sci"]},
          {"id":"ZX218","types":["std"]},{"id":"ZX217","types":["std"]},{"id":"ZX219","types":["pe"]},
          {"id":"ZX306","types":["std"]},{"id":"ZX319","types":["std"]},{"id":"ZX208","types":["std"]},
          {"id":"ZX207","types":["std"]},{"id":"ZX203","types":["std"]},{"id":"ZX204","types":["std"]},
          {"id":"ZX205","types":["std"]},{"id":"ZX206","types":["std"]}],
 "courses":{"English":{"teacher":"T-Bond","hours":5},"Chinese":{"teacher":"T-Lu","hours":3},
   "Precalc":{"teacher":"T-Yuan","hours":4},"History":{"teacher":"T-Ikerd","hours":4},
   "PE":{"teacher":"T-OBrien","hours":2,"type":"pe"},"CreWriting":{"teacher":"T-Mahaffey","hours":2},
   "Japanese":{"teacher":"T-Gong","hours":1},"PubSpeaking":{"teacher":"T-Voss","hours":1},
   "CAS":{"teacher":"T-Zhang","hours":2},"Physics":{"teacher":"T-Xu","hours":3,"type":"sci"},
   "Chemistry":{"teacher":"T-Chen","hours":3,"type":"sci"},"Biology":{"teacher":"T-Bai","hours":3,"type":"sci"},
   "Computer":{"teacher":"T-He","hours":3},"Economics":{"teacher":"T-Econ","hours":3},
   "Geography":{"teacher":"T-Geo","hours":3},"Music":{"teacher":"T-Music","hours":3},
   "Art":{"teacher":"T-Art","hours":3}},
 "groups":[
  {"id":"10-A block","curriculum":[{"course":"English"},{"course":"Chinese"},{"course":"Precalc"},
    {"course":"History"},{"course":"PE"},{"course":"CreWriting"},{"course":"Japanese"},
    {"course":"PubSpeaking"},{"course":"CAS","teacher":"T-Zhang"},
    {"pool":"SCI","choose":2,"options":["Physics","Chemistry","Biology"],"chosen":["Physics","Chemistry"]},
    {"pool":"ELE","choose":1,"options":["Computer","Economics","Geography","Music","Art"],"chosen":["Computer"]}]},
  {"id":"10-B block","curriculum":[{"course":"English"},{"course":"Chinese"},{"course":"Precalc"},
    {"course":"History"},{"course":"PE"},{"course":"CreWriting"},{"course":"Japanese"},
    {"course":"PubSpeaking"},{"course":"CAS","teacher":"T-Zhang-B"},
    {"pool":"SCI","choose":2,"options":["Physics","Chemistry","Biology"],"chosen":["Chemistry","Biology"]},
    {"pool":"ELE","choose":1,"options":["Computer","Economics","Geography","Music","Art"],"chosen":["Computer"]}]}]}

# Convenient global references; convert unavailable slots from lists to hashable tuples in sets.
ROOMS=DATA["rooms"]; UN={t:{tuple(x) for x in v} for t,v in DATA["unavail"].items()}

# Expand each curriculum entry into one record for every required lesson session.
def lessons(D):
    # L will hold one dictionary per individual lesson.
    L=[]
    for g in D["groups"]:
        for e in g["curriculum"]:
            # Use the selected courses from a pool, or the single named course for a normal entry.
            for c in (e.get("chosen") or [e["course"]]):
                co=D["courses"][c]
                # Create unique lesson IDs and store group, course, teacher, and room type.
                L+=[dict(id=f"{g['id']}_{c}_{i}",g=g["id"],c=c,t=e.get("teacher") or co["teacher"],
                         ty=co.get("type","std")) for i in range(co["hours"])]
    return L

# Validate a completed schedule; return a list of errors (an empty list means it passes).
def check(S,L):
    """All hard constraints + hours/pools. Empty list == feasible."""
    # First check that the schedule contains exactly the expected lesson IDs.
    e=[] if set(S)=={l["id"] for l in L} else ["missing lessons"]
    # Counters track clashes, daily group/teacher loads, and consecutive-block slots.
    seen,gd,td,cw=dd(int),dd(int),dd(int),dd(list)
    for l in L:
        d,p,r=S[l["id"]]
        # Enforce Friday’s five-period limit.
        if d==5 and p>5: e.append(f"{l['id']} Fri P{p}")
        # Ensure restricted courses use periods of the required duration.
        if p not in CP.get(l["c"],ALLP): e.append(f"{l['id']} {l['c']} wrong period length")
        # CAS must be in periods 6–7 on Monday through Thursday.
        if l["c"]=="CAS" and not(d<5 and p in(6,7)): e.append(f"{l['id']} CAS window")
        # A teacher cannot be scheduled during a listed unavailable slot.
        if (d,p) in UN.get(l["t"],()): e.append(f"{l['id']} teacher unavailable")
        if l["c"] in BLOCK: cw[l["g"],l["c"]]+=[(d,p)]
        # Count teacher, room, and group use at this day/period to detect clashes.
        seen["t",l["t"],d,p]+=1; seen["r",r,d,p]+=1; seen["g",l["g"],d,p]+=1
        gd[l["g"],d]+=1; td[l["t"],d]+=1
    # Any resource used more than once in a slot creates a clash error.
    e+=[f"clash {k}" for k,v in seen.items() if v>1]
    # Each group must have seven lessons Monday–Thursday and five on Friday.
    e+=[f"day load {k[0]} d{k[1]}={v}" for k,v in gd.items() if v!=(7 if k[1]<5 else 5)]
    # A teacher may teach no more than CAP lessons on any day.
    e+=[f"teacher load {k}" for k,v in td.items() if v>CAP]
    # Verify block courses occur on one day in consecutive periods.
    for k,v in cw.items():
        v.sort()
        if len({d for d,_ in v})!=1 or [p for _,p in v]!=list(range(v[0][1],v[0][1]+len(v))): e.append(f"block {k}")
    # Both groups must schedule CAS on the same day.
    if len({S[l["id"]][0] for l in L if l["c"]=="CAS"})>1: e.append("CAS days differ across blocks")
    # Count lessons per group/course to verify required hours and course-pool selections.
    h=dd(int)
    for l in L: h[l["g"],l["c"]]+=1
    # Also export a separate CSV and print a grid for each group.
    for g in DATA["groups"]:
        if h[g["id"],"PubSpeaking"]!=1: e.append(f"{g['id']} PubSpeaking!=1")
        if h[g["id"],"PE"]!=2: e.append(f"{g['id']} PE!=2")
        for en in g["curriculum"]:
            if "pool" in en:
                got={l["c"] for l in L if l["g"]==g["id"] and l["c"] in en["options"]}
                if got!=set(en["chosen"]) or len(en["chosen"])!=en["choose"]: e.append(f"{g['id']} pool {en['pool']}")
            elif h[g["id"],en["course"]]!=DATA["courses"][en["course"]]["hours"]: e.append(f"{g['id']} hours {en['course']}")
    return e

# Calculate soft-constraint scores; lower total scores represent preferable timetables.
def metrics(S,L):
    # Gather group periods/rooms, teacher periods, and course counts per group/day.
    gd,td,cd=dd(list),dd(list),dd(int)
    for l in L:
        d,p,r=S[l["id"]]
        gd[l["g"],d]+=[(p,r)]; td[l["t"],d]+=[p]
        if l["c"] not in BLOCK: cd[l["g"],l["c"],d]+=1
    # Count room changes between adjacent periods for each group.
    mv=sum(1 for v in gd.values() for a,b in zip(sorted(v),sorted(v)[1:]) if b[0]==a[0]+1 and a[1]!=b[1])
    # Count empty periods between a teacher’s lessons on the same day.
    gp=sum(b-a-1 for v in td.values() for a,b in zip(sorted(v),sorted(v)[1:]))
    # Penalize extra occurrences of the same non-block course on one day.
    sm=sum(n-1 for n in cd.values() if n>1)
    # Build each group’s daily lesson-count list for the balance calculation.
    ct=dd(lambda:[0]*DAYS)
    for (g,d),v in gd.items(): ct[g][d-1]=len(v)
    # Sum squared deviations from each group’s average daily lesson count.
    bl=sum(sum((x-sum(v)/DAYS)**2 for x in v) for v in ct.values())
    # Combine the metrics with weights into one score to minimize.
    return dict(move=mv,gap=gp,same=sm,bal=round(bl,1),total=2*mv+gp+3*sm+bl)

# Assign a compatible classroom to every scheduled lesson; return None if no room fits.
def rooms(st,L):
    # Group lessons by time slot so simultaneous classes receive different rooms.
    bs=dd(list)
    for l in L: bs[st[l["g"]][l["id"]]]+=[l]
    S={}
    for s,ls in bs.items():
        # Rooms already used in this slot cannot be assigned again.
        u=set()
        for l in sorted(ls,key=lambda l:PRI[l["ty"]]):
            # Choose the first unused room that supports this lesson’s required type.
            r=next((r["id"] for r in ROOMS if l["ty"] in r["types"] and r["id"] not in u),None)
            if not r: return None
            u.add(r); S[l["id"]]=(*s,r)
    return S

# Construct one group’s schedule, respecting restrictions imposed by the other group.
def build(gid,L,forb,trem,pef,seed,casday=None):
    # Seeded randomness makes each construction attempt reproducible.
    rnd=random.Random(seed); free=set(SLOTS); S={}; dc=dd(int); tc=dd(int)
    # Group this group’s lesson records by course.
    byc=dd(list)
    for l in L:
        # Put the course name (up to nine characters) into its scheduled grid cell.
        if l["g"]==gid: byc[l["c"]]+=[l]
    # Helper to place a multi-period unit and update available slots and daily counters.
    def put(unit,t,slots):
        for l,s in zip(unit,slots): S[l["id"]]=s
        free.difference_update(slots); dc[slots[0][0]]+=len(slots); tc[t,slots[0][0]]+=len(slots)
    # Enumerate valid candidate windows for the consecutive CAS and Creative Writing blocks.
    wins={"CAS":[[(d,6),(d,7)] for d in ([casday] if casday else range(1,5))],
          "CreWriting":[[(d,p),(d,p+1)] for d in range(1,6) for p in range(1,7 if d<5 else 5)]}
    # Place block courses first because their consecutive-period constraints are restrictive.
    for name in ("CAS","CreWriting"):
        t=byc[name][0]["t"]
        c=[w for w in wins[name] if set(w)<=free and trem[t,w[0][0]]-tc[t,w[0][0]]>=2
           and all(forb.get(s)!=t and s not in UN.get(t,()) for s in w)]
        if not c: return None
        put(byc[name],t,rnd.choice(c))
    # Collect all remaining lessons that can be placed in individual slots.
    sing=[l for v in byc.values() for l in v if l["c"] not in BLOCK]
    # A slot is acceptable only if teacher, period, PE, and daily-capacity rules permit it.
    ok=lambda l,s: (forb.get(s)!=l["t"] and s not in UN.get(l["t"],())
                    and s[1] in CP.get(l["c"],ALLP)
                    and not(l["c"]=="PE" and s in pef)
                    and trem[l["t"],s[0]]-tc[l["t"],s[0]]>0)
    # Repeatedly schedule the remaining lesson with the fewest available choices first.
    while sing:
        # Count valid slots for each remaining lesson (most-constrained-first heuristic).
        n={l["id"]:sum(ok(l,s) for s in free) for l in sing}
        if min(n.values())==0: return None
        l=min(sing,key=lambda l:(n[l["id"]],rnd.random()))
        # Prefer days with fewer group lessons, then fewer lessons for that teacher; randomize ties.
        s=min((s for s in free if ok(l,s)),key=lambda s:(dc[s[0]],tc[l["t"],s[0]],rnd.random()))
        S[l["id"]]=s; free.discard(s); dc[s[0]]+=1; tc[l["t"],s[0]]+=1; sing.remove(l)
    return S,dict(tc)

# Build 10-A first, then search for a compatible 10-B timetable.
def pair(L):
    # Try multiple seeded constructions for group A.
    for sa in range(1,9):
        r=build("10-A block",L,{},dd(lambda:CAP),set(),sa)
        if not r: continue
        A,uA=r
        # Extract the day on which group A has CAS so group B can use the same day.
        cd=A[next(l["id"] for l in L if l["g"]=="10-A block" and l["c"]=="CAS")][0]
        # Record group A’s teacher/slot assignments to prevent same-teacher clashes in group B.
        forb={A[l["id"]]:l["t"] for l in L if l["g"]=="10-A block"}
        # Group B must not schedule PE in group A’s PE slots.
        pef={A[l["id"]] for l in L if l["g"]=="10-A block" and l["c"]=="PE"}
        # Calculate each teacher’s remaining daily capacity after group A is scheduled.
        tr=dd(lambda:CAP); tr.update({k:CAP-v for k,v in uA.items()})
        # Try up to 40 seeded constructions for group B for each group-A candidate.
        for sb in range(1,41):
            r2=build("10-B block",L,forb,tr,pef,sb,cd)
            if r2: return A,r2[0]
    return None

# Improve a valid timetable using simulated annealing; return the best schedule and score found.
def opt(st,L,it=6000,seed=0):
    rnd=random.Random(seed); cur={g:dict(v) for g,v in st.items()}
    co={l["id"]:l["c"] for l in L}
    # Score the initial timetable and save it as the best solution so far.
    cc=metrics(rooms(cur,L),L)["total"]; best,bc=copy.deepcopy(cur),cc
    # Each iteration proposes a change; temperature gradually decreases over time.
    for i in range(it):
        # Higher temperature early on makes accepting worse moves more likely.
        T=1.5*(1-i/it)+.01; g=rnd.choice(list(cur)); ids=list(cur[g])
        sg=[x for x in ids if co[x] not in BLOCK]; old=copy.deepcopy(cur[g]); mv=True
        if rnd.random()<.6:
            a,b=rnd.sample(sg,2); cur[g][a],cur[g][b]=cur[g][b],cur[g][a]
        # Otherwise, try relocating a two-period block into two adjacent ordinary-lesson slots.
        else:
            u=rnd.choice(["CAS","CreWriting"])
            ul=sorted((x for x in ids if co[x]==u),key=lambda x:cur[g][x][1])
            x,y=rnd.sample(sg,2); sx,sy=cur[g][x],cur[g][y]
            mv=sx[0]==sy[0] and abs(sx[1]-sy[1])==1
            if mv:
                d,p=sx[0],min(sx[1],sy[1]); mv=not(u=="CAS" and not(d<5 and p==6))
            if mv:
                if sx[1]>sy[1]: x,y=y,x
                o1,o2=cur[g][ul[0]],cur[g][ul[1]]
                cur[g][ul[0]],cur[g][ul[1]]=(d,p),(d,p+1); cur[g][x],cur[g][y]=o1,o2
        # Only evaluate proposals that pass the preliminary move checks.
        if mv:
            nf=rooms(cur,L)
            # Reassign rooms and accept only schedules that pass hard-constraint validation.
            if nf and not check(nf,L):
                nc=metrics(nf,L)["total"]
                # Always accept an improved score; sometimes accept a worse one to escape local optima.
                if nc<=cc or rnd.random()<math.exp(-(nc-cc)/T):
                    cc=nc
                    if nc<bc: best,bc=copy.deepcopy(cur),nc
                    continue
        # If the proposal was invalid or rejected, restore this group’s previous schedule.
        cur[g]=old
    return best,bc

# Print one group’s timetable as a weekday-by-period grid.
def show(S,L,gid):
    # Initialize every displayed cell as empty, represented by a dot.
    gr={(d,p):"." for d in range(1,6) for p in range(1,8)}
    for l in L:
        if l["g"]==gid: gr[S[l["id"]][:2]]=l["c"][:9]
    print(f"-- {gid} "+"-"*66)
    print("  P time      | "+" | ".join(f"{h:^9}" for h in ("Mon","Tue","Wed","Thu","Fri")))
    for p in range(1,8):
        print(f"  {p} {TIMES[p-1]} | "+" | ".join(f"{gr[d,p]:^9}" for d in range(1,6)))

# Normal run: build schedules, optimize them, report scores, export CSVs, and print grids.
def main():
    # Convert curriculum to individual lessons, then construct a compatible baseline timetable.
    L=lessons(DATA); pr=pair(L)
    if not pr: print("CRITICAL: construction failed"); return
    st={"10-A block":pr[0],"10-B block":pr[1]}
    # Keep the baseline for comparison and run optimization with three different seeds.
    base=rooms(st,L); runs=[opt(st,L,seed=s) for s in (1,2,3)]
    # Select the optimization run with the lowest score and assign its final rooms.
    os_,_=min(runs,key=lambda x:x[1]); O=rooms(os_,L)
    mb,mo=metrics(base,L),metrics(O,L)
    # Report the number of lessons and whether the final timetable passes validation.
    print("lessons:",len(L),"| validation:",check(O,L) or "PASS")
    print(f"{'metric':<7}{'baseline':>9}{'optimized':>10}")
    for k in ("move","gap","same","bal","total"): print(f"{k:<7}{mb[k]:>9}{mo[k]:>10}")
    # Export all groups into one combined CSV file.
    with open("timetable.csv","w",newline="") as f:
        w=csv.writer(f); w.writerow(["block","day","period","course","teacher","room"])
        for l in sorted(L,key=lambda l:(l["g"],)+O[l["id"]][:2]):
            d,p,r=O[l["id"]]; w.writerow([l["g"],d,p,l["c"],l["t"],r])
    for g in DATA["groups"]:
        with open(f"timetable_{g['id'].replace(' ','_')}.csv","w",newline="") as f:
            w=csv.writer(f); w.writerow(["day","period","time","course","teacher","room"])
            for l in sorted((x for x in L if x["g"]==g["id"]),key=lambda x:O[x["id"]][:2]):
                d,p,r=O[l["id"]]; w.writerow([d,p,TIMES[p-1],l["c"],l["t"],r])
        show(O,L,g["id"])
    print("files: timetable.csv, timetable_10-A_block.csv, timetable_10-B_block.csv")

# Test mode: assert that construction, validation, optimization, and key rules work.
def selftest():
    # Assert that a pair of schedules can be built successfully.
    L=lessons(DATA); pr=pair(L); assert pr
    st={"10-A block":pr[0],"10-B block":pr[1]}
    # Assert that room assignment succeeds and the baseline has no validation errors.
    S=rooms(st,L); assert S and not check(S,L)
    # Run a shorter optimization pass and validate its output.
    o,c=opt(st,L,it=3000,seed=1); O=rooms(o,L)
    assert O and not check(O,L) and c<=metrics(S,L)["total"]
    assert len({O[l["id"]][0] for l in L if l["c"]=="CAS"})==1
    for gid in ("10-A block","10-B block"):
        assert sum(1 for l in L if l["g"]==gid and l["c"]=="PE")==2, "exactly 2 PE per block"
    for l in L:
        if l["c"]=="Japanese": assert O[l["id"]][1] in JP
        if l["c"]=="PE": assert O[l["id"]][1] in P55, "PE must be the 55-min afternoon period (P5)"
    print("selftest OK")

# Run tests when --test is supplied; otherwise execute the normal timetable generator.
selftest() if "--test" in sys.argv else main()
