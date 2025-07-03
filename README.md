# working-with-alvis-from-ki

Hopefully, helpful info for KI researchers, about getting started working with AI on the alvis supercomputer.

## Get Access

Apply for acess at https://supr.naiss.se/

You need to be a at least at the level of phd or ask your superviors to be granted the default allocation.

## Caveats

### Working remotly

When at KI or connected to eduroam you ..
VPN
## Gettings started



sinfo --Node --states=idle,mix --noheader \
      -O NodeList:12,StateCompact:8,Gres:90,GresUsed:90,CPUs:21,Memory:10 |
awk 'BEGIN { printf "%-12s %-8s %-35s %-35s %21s %10s\n", \
                   "NODE","STATE","GRES","GRES-USED","CPUs","MEM(MB)" }
     $2!="plnd" && $3~/gpu:A100/ {
                   sub(/\(.*/,"",$3); sub(/\(.*/,"",$4);
                   printf "%-12s %-8s %-35s %-35s %21s %10s\n", \
                          $1,$2,$3,$4,$5,$6 }' |
column -t
