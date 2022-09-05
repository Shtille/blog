---
layout: post
title: "ESO DoTs comparison"
author: "Shtille"
categories: journal
tags: [C++,samples,ESO]
#image: cards.jpg
---

ESO is an abbreviature for Elder Scrolls Online™ computer game. This article will describe how to compare  damage over time effects (DoTs) performance for making better skill rotation.

## Skills comparison formula

This place is reserved for future description.

## Test program for dots comparison

I wrote a simple program that compares hardcoded DoTs DPS.

### Program code

Skill names you can check at [ESO skillbook](https://eso-skillbook.com/skilltree/templar).

```cpp
#include <string>
#include <iostream>

class Skill {
public:
	// The initial idea is to create few methods to create different kinds of skills
	static Skill Create(
		const std::string& name,
		double momental, 
		double damage_tick, 
		double time_tick,
		double damage_over_duration,
		double duration,
		double prepare_time = 0.)
	{
		return Skill(
			name,
			momental, 
			damage_tick, 
			time_tick,
			damage_over_duration,
			duration,
			prepare_time);
	}
	double GetDamage() const
	{
		return momental 
			+ (damage_tick / time_tick) * duration 
			+ damage_over_duration;
	}
	double GetDuration() const
	{
		return duration + prepare_time;
	}
	double GetDps() const
	{
		return GetDamage() / GetDuration();
	}
	const std::string& GetName() const
	{
		return name;
	}

protected:
	Skill(
		const std::string& name,
		double momental, 
		double damage_tick, 
		double time_tick,
		double damage_over_duration,
		double duration,
		double prepare_time)
	: name(name)
	, momental(momental)
	, damage_tick(damage_tick)
	, time_tick(time_tick)
	, damage_over_duration(damage_over_duration)
	, duration(duration)
	, prepare_time(prepare_time)
	{
	}
private:
	std::string name;
	double momental;
	double damage_tick;
	double time_tick;
	double damage_over_duration;
	double duration;
	double prepare_time;
};

Skill g_skills[] = {
	Skill::Create("[0] Biting Jabs",   4276.*3.,    0., 1.,     0.,  1.     ),
	Skill::Create("[1] Deadly Cloak",        0., 2505., 2.,     0., 20.     ),
	Skill::Create("[2] Blazing Spear",    9530.,  803., 1.,     0., 10.     ),
	Skill::Create("[3] Barbed Trap",      5348.,    0., 1., 17280., 20., 1.5),
	Skill::Create("[4] Solar Barrage",       0., 1865., 2.,     0., 22.     ),
};

/**
 * Function compares skill sets (1,2) and (1,3).
 * @param[in] skill1 		The base skill (spammable).
 * @param[in] skill2 		The skill to test from skillset (1,2).
 * @param[in] skill3 		The skill to test from skillset (1,3).
 * @return Returns true if skillset (1,3) is better than (1,2), and false otherwise.
 */
bool CompareSkillSets(const Skill& skill1, const Skill& skill2, const Skill& skill3)
{
	// Define variables
	double T1 = skill1.GetDuration();
	double T2 = skill2.GetDuration();
	double T3 = skill3.GetDuration();
	double R1 = skill1.GetDps();
	double R2 = skill2.GetDps();
	double R3 = skill3.GetDps();
	bool result = (R3-R2)+((T3-T2)*(R1*T1)/(T2*T3)) > 0.;
	const Skill *first, *second;
	if (result)
		second = &skill2, first = &skill3;
	else
		first = &skill2, second = &skill3;
	std::cout << "skill " << first->GetName() << 
		" is better than " << second->GetName() << std::endl;
	return result;
}

int main(int argc, char const *argv[])
{
	CompareSkillSets(g_skills[0], g_skills[1], g_skills[2]);
	CompareSkillSets(g_skills[0], g_skills[2], g_skills[3]);
	CompareSkillSets(g_skills[0], g_skills[1], g_skills[3]);
	//---
	std::cout << std::endl;
	//---
	CompareSkillSets(g_skills[0], g_skills[2], g_skills[3]);
	CompareSkillSets(g_skills[0], g_skills[3], g_skills[4]);
	CompareSkillSets(g_skills[0], g_skills[2], g_skills[4]);
	return 0;
}
```

### Program output

```
skill [1] Deadly Cloak is better than [2] Blazing Spear
skill [2] Blazing Spear is better than [3] Barbed Trap
skill [1] Deadly Cloak is better than [3] Barbed Trap

skill [2] Blazing Spear is better than [3] Barbed Trap
skill [3] Barbed Trap is better than [4] Solar Barrage
skill [2] Blazing Spear is better than [4] Solar Barrage
```

### Conclusion

```
[3] Barbed Trap < [2] Blazing Spear < [1] Deadly Cloak
[4] Solar Barrage < [3] Barbed Trap < [2] Blazing Spear
[4] Solar Barrage < [1] Deadly Cloak
```

We placed initially skills in the perfect order of DPS descendance

### Improvements

- We should use inheritance to describe the skills because of the different behaviour.
- Make large list of skills and sort it in DPS descendance order.